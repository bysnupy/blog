# Introduction to using Multus on OpenShift

## Summary

OpenShift allows a pod attach multiple network interfaces using Multus CNI that chains multiple plugins through the `NetworkAttachmentDefinition` object.
Note that there are multiple network interface, IPAM(IP address management) and other options, and we may use the network options appropriately for your use cases.
In this article, I will briefly walk through using Multus on OpenShift focused expected usual use cases using the Main and IPAM plugins.

## Using Multus on OpenShift

With OpenShift, Multus CNI is managed multiple networks by the Cluster Network Operator(CNO) using CustomResource(CR), when you specify an additional network to the CR, the CNO generates the `NetworkAttachmentDefinition` object automatically for Multus CNI.

![multus_and_cno_operator](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_cno_operator_figure1.png)

## Bridge for communication all containers on the same node host

The bridge main plugin will create linux bridge interface without linking physical host interface and forwarding rules for external network, so it is desired container-to-container communication on the same node host.

![multus_bridge](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_bridge_figure2.png)

The bridge plugin usually would work well with host-local and whereabouts IPAM plugins. For the following example, it's configured with whereabouts for assigning IP addresses dynamically across the cluster, but it does not mean a pod can access to another pod on the other node host through attached net1 interface.

```console
$ oc edit networks.operator.openshift.io cluster
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks: 
  - name: bridge-main
    namespace: multus-test
    type: Raw
    rawCNIConfig: '{
      "cniVersion": "0.3.1",
      "name": "bridge-main",
      "type": "bridge",
      "bridge": "mybr0",
      "ipam": {
          "type": "whereabouts",
          "range": "192.168.12.0/24",
          "range_start": "192.168.12.10",
          "range_end": "192.168.12.200",
          "exclude": [
              "192.168.12.1/32",
              "192.168.12.254/32"
          ]
      }
    }'
    ...snip...
```

`NetworkAttachmentDefinition` defined in the above CR will be created in the specified namespace as follows.

```console
$ oc get net-attach-def -n multus-test
NAME          AGE
bridge-main   24m
```

There are two pods in the same node host, and check if the bridge is configured correctly using `ip link` command on the node host.
```console
// Check the pods IP addresses after specifying network you configured.
$ oc patch deploy/pod-a \
  --patch '{"spec":{"template":{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"bridge-main"}}}}}'
$ oc patch deploy/pod-b \
  --patch '{"spec":{"template":{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"bridge-main"}}}}}'

$ oc get pod -o wide -n multus-test
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE                         NOMINATED NODE   READINESS GATES
pod-a-5bb7694d4-j8g6b    1/1     Running   0          2m42s   10.128.2.20   worker1.ocp46rt.priv.local   <none>           <none>
pod-b-85ffcd4c7c-xpp2z   1/1     Running   0          2m42s   10.128.2.21   worker1.ocp46rt.priv.local   <none>           <none>

// Access the node host running the pods.
$ oc debug node/worker1.ocp46rt.priv.local
:
sh-4.4# chroot /host

// List bridge interfaces.
sh-4.4# link show type bridge
18: mybr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 42:db:f8:c4:5e:37 brd ff:ff:ff:ff:ff:ff

// List all the connected ports.
sh-4.4# ip link show master mybr0
48: veth0284b088@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mybr0 state UP mode DEFAULT group default 
    link/ether da:a3:29:82:6f:2e brd ff:ff:ff:ff:ff:ff link-netns 1189a176-2644-4756-b26d-61680807a0de
49: vethc605f6cf@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mybr0 state UP mode DEFAULT group default 
    link/ether 42:db:f8:c4:5e:37 brd ff:ff:ff:ff:ff:ff link-netns 9fa20465-69d9-4d38-ba30-10e770f6aed1
```

Test if the pods can communicate each other using `ip netns exec` command as follows.
```console
// Check IP addresses, in this case, it's about "pod-b".
sh-4.4# ip netns exec 1189a176-2644-4756-b26d-61680807a0de ip -4 a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if47: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default  link-netnsid 0
    inet 10.128.2.21/23 brd 10.128.3.255 scope global eth0
       valid_lft forever preferred_lft forever
5: net1@if48: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default  link-netnsid 0
    inet 192.168.12.11/24 brd 192.168.12.255 scope global net1
       valid_lft forever preferred_lft forever

// Check IP addresses, in this case, it's about "pod-a".
sh-4.4# ip netns exec 9fa20465-69d9-4d38-ba30-10e770f6aed1 ip -4 a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if46: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default  link-netnsid 0
    inet 10.128.2.20/23 brd 10.128.3.255 scope global eth0
       valid_lft forever preferred_lft forever
5: net1@if49: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default  link-netnsid 0
    inet 192.168.12.14/24 brd 192.168.12.255 scope global net1
       valid_lft forever preferred_lft forever

// Ping from "pod-a" to "pod-b".
sh-4.4# ip netns exec 9fa20465-69d9-4d38-ba30-10e770f6aed1 ping -c3 192.168.12.11
PING 192.168.12.11 (192.168.12.11) 56(84) bytes of data.
64 bytes from 192.168.12.11: icmp_seq=1 ttl=64 time=0.064 ms
64 bytes from 192.168.12.11: icmp_seq=2 ttl=64 time=0.095 ms
64 bytes from 192.168.12.11: icmp_seq=3 ttl=64 time=0.062 ms

--- 192.168.12.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 86ms
rtt min/avg/max/mdev = 0.062/0.073/0.095/0.018 ms
sh-4.4#
```

We verified the bridge and whereabouts plugins are configured as expected.

## Host-device for assigning dedicated host network device

The host-device plugin moves the specified device on the host into the same network namespace of only a pod, so the moved device cannot share with other pods.
So you can consider to use this if your workload is required a dedicated physical interface.

![multus_host_device](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_host_device_figure3.png)

The attached interface keeps the physical mac address and it should not be duplicated IP addresses with the connected network,
so you can use static, whereabouts and dhcp IPAMs. This time I will demonstrate host-device with dhcp IPAM, and you prepare a DHCP server in advance.

```console
$ oc edit networks.operator.openshift.io cluster
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks: 
  - name: host-device-main
    namespace: multus-test
    type: Raw
    rawCNIConfig: '{
      "cniVersion": "0.3.1",
      "name": "host-device-main",
      "type": "host-device",
      "device": "ens8",
      "ipam": {
          "type": "dhcp"
      }
    }'
    ...snip...
```

`NetworkAttachmentDefinition` defined in the above CR will be added as follows.

```console
$ oc get net-attach-def -n multus-test
NAME               AGE
bridge-main        22h
host-device-main   19m
```

The two pods will run on the following worker nodes, and the specified ens8 MAC address is as follows on each node host.
```console
// worker1
ens8: 00:1a:4a:16:06:76

// worker2
ens8: 00:1a:4a:16:06:81
```
After specifying `NetworkAttachmentDefinition` to each pod, you can see redeployed pods as follows.
```console
// Check the pods IP addresses after specifying network you configured.
$ oc patch deploy/pod-a \
  --patch '{"spec":{"template":{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"host-device-main"}}}}}'
$ oc patch deploy/pod-b \
  --patch '{"spec":{"template":{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"host-device-main"}}}}}'

$  oc get pod -o wide -n multus-test
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                         NOMINATED NODE   READINESS GATES
pod-a-66b77fb7f6-wbx5f   1/1     Running   0          56m   10.128.2.15   worker1.ocp46rt.priv.local   <none>           <none>
pod-b-cc754dd7-pq9km     1/1     Running   0          42m   10.130.2.13   worker2.ocp46rt.priv.local   <none>           <none>
```

You can also see DHCPOFFER logs for the pods at the DHCP server.
If there are no dhcp-daemon DaemonSet pods in the "openshift-multus", when the dhcp IPAM is configured, it makes the pods run either.
```console
// for "pod-a"
DHCPDISCOVER(ens11) 00:1a:4a:16:06:76
DHCPOFFER(ens11) 192.168.12.20 00:1a:4a:16:06:76
DHCPREQUEST(ens11) 192.168.12.20 00:1a:4a:16:06:76
DHCPACK(ens11) 192.168.12.20 00:1a:4a:16:06:76
:
// for "pod-b"
DHCPDISCOVER(ens11) 00:1a:4a:16:06:81
DHCPOFFER(ens11) 192.168.12.21 00:1a:4a:16:06:81
DHCPREQUEST(ens11) 192.168.12.21 00:1a:4a:16:06:81
DHCPACK(ens11) 192.168.12.21 00:1a:4a:16:06:81

# oc get pod -o wide -n openshift-multus
NAME                                    READY   STATUS    RESTARTS   AGE     IP             NODE                         NOMINATED NODE   READINESS GATES
pod/dhcp-daemon-5g4vq                   1/1     Running   0          1h12m   192.168.9.35   worker1.ocp46rt.priv.local   <none>           <none>
pod/dhcp-daemon-5w9kj                   1/1     Running   0          1h12m   192.168.9.36   worker2.ocp46rt.priv.local   <none>           <none>
:
```

Check netns(network namespace) for running pod using `crictl inspectp` command on the node host first.

```console
// Access the node host running the pods.
$ oc debug node/worker1.ocp46rt.priv.local
:
sh-4.4# chroot /host

sh-4.4# crictl pods 
POD ID              CREATED             STATE               NAME                            NAMESPACE                                ATTEMPT
352feb5afa9ca       2 hours ago         Ready               pod-a-66b77fb7f6-wbx5f          multus-test                              0
:

sh-4.4# crictl inspectp  --output json 352feb5afa9ca | \
        jq '.info.runtimeSpec.linux.namespaces | .[] | select(.type == "network")'
{
  "type": "network",
  "path": "/var/run/netns/46775c43-417c-4b84-891b-0053bc45c006"
}
sh-4.4#
```

Test if the pods can communicate with web server in the external network using `ip netns exec` command using above netns as follows.
```console
// You can see the both pod IP and attached net1 IP addresses with MAC address.
sh-4.4# ip netns exec 46775c43-417c-4b84-891b-0053bc45c006 ip a                       
:
3: eth0@if27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 0a:58:0a:80:02:0f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.128.2.15/23 brd 10.128.3.255 scope global eth0
       valid_lft forever preferred_lft forever
:
4: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:1a:4a:16:06:76 brd ff:ff:ff:ff:ff:ff
    inet 192.168.12.20/24 brd 192.168.12.255 scope global net1
       valid_lft forever preferred_lft forever
:

// Web server IP was 192.168.12.10 and 8080 port was listening, if accessed, return "Web server Running" message.
sh-4.4# ip netns exec 46775c43-417c-4b84-891b-0053bc45c006 curl -vs http://192.168.12.10:8080/
:
< HTTP/1.1 200 OK
< Date: Fri, 01 Jan 2021 08:32:48 GMT
< Server: Apache/2.4.37 (Red Hat Enterprise Linux)
:
< 
Web server Running
* Connection #0 to host 192.168.12.10 left intact

// And the source IP was 192.168.12.20 in the web server as follows.
192.168.12.20 - - ... "GET / HTTP/1.1" 200 11 "-" "curl/7.61.1"
```

The pod can access the other pod either using IP address.
```
sh-4.4# ip netns exec 46775c43-417c-4b84-891b-0053bc45c006 curl -vs http://192.168.12.21:8080/
:
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/2.7.5
:
< 
Pod B
* Closing connection 0
sh-4.4#
```
We verified the host-device and dhcp plugins are configured as expected. If you use static plugin instead of dhcp, external DHCP server would not be required, but you should add each `NetworkAttachmentDefinition` for each network device with unique IP address across cluster.

## MACvlan plugin for multiple interfaces with different MAC addresses on top of a single interface

You can bind and share a physical interface that is associated with multiple macvlan interfaces directly without a bridge.
And each macvlan has its own MAC address, it makes it easy to use additional network on multiple pods.

![multus_macvlan](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_macvlan_figure4.png)

The macvlan is useful when you use additional networks usually, attached each interface by macvlan on the pods would work like each physical interface, but you need not to prepare physical interfaces per the pod count.

