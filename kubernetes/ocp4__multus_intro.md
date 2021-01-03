# Introduction to using Multus on OpenShift

## Summary

OpenShift allows a pod attach multiple network interfaces using Multus CNI that chains multiple plugins through the `NetworkAttachmentDefinition` object.
Note that there are multiple network interface, IPAM(IP address management) and other options, and we may use the network options appropriately for your use cases.
In this article, I will briefly walk through using Multus on OpenShift focused expected usual use cases using the Main and IPAM plugins.

## Using Multus on OpenShift

With OpenShift, Multus CNI is managed multiple networks by the Cluster Network Operator(CNO) using CustomResource(CR), when you specify an additional network to the CR, the CNO generates the `NetworkAttachmentDefinition` object automatically for Multus CNI.

![multus_and_cno_operator](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_cno_operator_figure1.png)

Be careful for the following access flow diagram. It is simplified intentionally, even though the SDN is connected through the host network using VXLAN tunneling in realty. It's skipped the details for focusing Multus access flow.

## bridge plug-in for communication all containers on the same node host

The bridge main plugin will create linux bridge interface without linking physical host interface and forwarding rules for external network, so it is desired container-to-container communication on the same node host.

![multus_bridge](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_bridge_flow_figure2.png)

The bridge plugin usually would work well with host-local and whereabouts IPAM plugins without external equipment(e.g. DHCP server).
For the following example, it's configured with host-local for assigning IP addresses dynamically on the same host.

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
          "type": "host-local",
          "subnet": "192.168.12.0/24",
          "rangeStart": "192.168.12.10",
          "rangeEnd": "192.168.12.200",
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
    inet 192.168.12.10/24 brd 192.168.12.255 scope global net1
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
    inet 192.168.12.11/24 brd 192.168.12.255 scope global net1
       valid_lft forever preferred_lft forever

// Ping from "pod-a" to "pod-b".
sh-4.4# ip netns exec 9fa20465-69d9-4d38-ba30-10e770f6aed1 ping -c3 192.168.12.10
PING 192.168.12.10 (192.168.12.10) 56(84) bytes of data.
64 bytes from 192.168.12.10: icmp_seq=1 ttl=64 time=0.064 ms
64 bytes from 192.168.12.10: icmp_seq=2 ttl=64 time=0.095 ms
64 bytes from 192.168.12.10: icmp_seq=3 ttl=64 time=0.062 ms

--- 192.168.12.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 86ms
rtt min/avg/max/mdev = 0.062/0.073/0.095/0.018 ms
sh-4.4#
```

We verified the bridge and whereabouts plugins are configured as expected.

## host-device plug-in for assigning dedicated host network device

With host-device plug-in, it allows a pod can bind underlay network directly using the host interface. It make the host-device plugin move a host interface into a pod network namespace during attached the interface to the pod, so the host interface will not be shown on the host until the pod stopped.

![multus_host_device](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_host_device_flow_figure3.png)

It's similar with macvlan, but host-device only map one host interface to one pod virtual interface, not using sub-interface. 
This plugin would work well with static, whereabouts and dhcp IPAMs. 
This time I will demonstrate host-device with dhcp IPAM, and you prepare a external DHCP server in advance.

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
NAME                                READY   STATUS    RESTARTS   AGE     IP             NODE                         NOMINATED NODE   READINESS GATES
dhcp-daemon-5g4vq                   1/1     Running   0          1h12m   192.168.9.35   worker1.ocp46rt.priv.local   <none>           <none>
dhcp-daemon-5w9kj                   1/1     Running   0          1h12m   192.168.9.36   worker2.ocp46rt.priv.local   <none>           <none>
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

The pod can access another "pod-b" either using IP address.
```console
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

## ipvlan plug-in for environment under limited MAC addresses
ipvlan plug-in may be used in case that limited MAC addresses affect the configuration, such as some switches device restrict the maximum number of mac address per physical port due to port security configuration.

![multus_ipvlan](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_ipvlan_flow_figure4.png)

Its sub-interface of the ipvlan can use distinct IP addresses with the same MAC address of the parent host interface. 
So it does not work well with DHCP server which depends on MAC address. 
Parent host interface acts as bridge(switch) with L2 mode, and it acts as router with L3 mode of the ipvlan plug-in.
With the following example, I will use whereabouts IPAM and L2 mode(default). Refer [IPVLAN Driver HOWTO](https://www.kernel.org/doc/Documentation/networking/ipvlan.txt) for more details.

```console
$ oc edit networks.operator.openshift.io cluster
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks: 
  - name: ipvlan-main
    namespace: multus-test
    type: Raw
    rawCNIConfig: '{
      "cniVersion": "0.3.1",
      "name": "ipvlan-main",
      "type": "ipvlan",
      "mode": "l2",
      "master": "ens9",
      "ipam": {
          "type": "whereabouts",
          "range": "192.168.12.0/24",
          "range_start": "192.168.12.50",
          "range_end": "192.168.12.200"
      }
    }'
    ...snip...
```

`NetworkAttachmentDefinition` defined in the above CR will be added as follows.

```console
$ oc get net-attach-def -n multus-test
NAME          AGE
ipvlan-main   64s
```

The two pods will run on the same worker nodes, and the specified ens9 MAC address is as follows. 
```console
// worker1
ens9: 00:1a:4a:16:06:77
```

After specifying `NetworkAttachmentDefinition` to each pod, you can see redeployed pods as follows.
```console
// Check the pods IP addresses after specifying network you configured.
$ oc patch deploy/pod-a \
  --patch '{"spec":{"template":{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"ipvlan-main"}}}}}'
$ oc patch deploy/pod-b \
  --patch '{"spec":{"template":{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"ipvlan-main"}}}}}'

$  oc get pod -o wide -n multus-test
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                         NOMINATED NODE   READINESS GATES
pod-a-7cd54fb895-ftcdl   1/1     Running   0          39s   10.128.2.20   worker1.ocp46rt.priv.local   <none>           <none>
pod-b-684c6bc666-6hss9   1/1     Running   0          38s   10.128.2.21   worker1.ocp46rt.priv.local   <none>           <none>
```

Check netns(network namespace) for running pod using `crictl inspectp` command on the node host first.

```console
// Access the node host running the pods.
$ oc debug node/worker1.ocp46rt.priv.local
:
sh-4.4# chroot /host

sh-4.4# crictl pods --name pod-a-7cd54fb895-ftcdl                      
POD ID              CREATED             STATE               NAME                     NAMESPACE           ATTEMPT
3f594489ab589       11 minutes ago      Ready               pod-a-7cd54fb895-ftcdl   multus-test         0

sh-4.4# crictl pods --name pod-b-684c6bc666-6hss9
POD ID              CREATED             STATE               NAME                     NAMESPACE           ATTEMPT
99c9343b01d54       11 minutes ago      Ready               pod-b-684c6bc666-6hss9   multus-test         0

sh-4.4# crictl inspectp  --output json {3f594489ab589,99c9343b01d54} | \
        jq '.info.runtimeSpec.linux.namespaces | .[] | select(.type == "network")'
{
  "type": "network",
  "path": "/var/run/netns/2c0c43c6-d5ad-46b0-9bca-1d27cc3868b7"
}
{
  "type": "network",
  "path": "/var/run/netns/eb9f2dbf-6872-4318-9ed0-d0bdf6da7f72"
}

sh-4.4#
```

Test if the pods can communicate with web server in the external network using `ip netns exec` command using above netns as follows.
```console
// You can see the both pod attached net1 IP addresses with the same MAC address.
sh-4.4# ip netns exec 2c0c43c6-d5ad-46b0-9bca-1d27cc3868b7 ip -4 a
:
3: eth0@if32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default  link-netnsid 0
    inet 10.128.2.20/23 brd 10.128.3.255 scope global eth0
:
4: net1@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default 
    inet 192.168.12.50/24 brd 192.168.12.255 scope global net1
:

sh-4.4# ip netns exec eb9f2dbf-6872-4318-9ed0-d0bdf6da7f72 ip a
:
3: eth0@if33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default  link-netnsid 0
    inet 10.128.2.21/23 brd 10.128.3.255 scope global eth0
:
4: net1@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default 
    inet 192.168.12.51/24 brd 192.168.12.255 scope global net1
:

// List ipvlan in the pod network namespace
sh-4.4# ip netns exec 2c0c43c6-d5ad-46b0-9bca-1d27cc3868b7 ip -d link show type ipvlan
4: net1@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 00:1a:4a:16:06:77 brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 68 maxmtu 65535                    <--- the same MAC
    ipvlan  mode l2 bridge addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 <--- L2 mode

sh-4.4# ip netns exec eb9f2dbf-6872-4318-9ed0-d0bdf6da7f72 ip -d link show type ipvlan
4: net1@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 00:1a:4a:16:06:77 brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 68 maxmtu 65535                    <--- the same MAC
    ipvlan  mode l2 bridge addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 <--- L2 mode

// ping from "pod-a" to "pod-b"
sh-4.4# ip netns exec 2c0c43c6-d5ad-46b0-9bca-1d27cc3868b7 ping -c1 192.168.12.51
PING 192.168.12.51 (192.168.12.51) 56(84) bytes of data.
64 bytes from 192.168.12.51: icmp_seq=1 ttl=64 time=0.139 ms

--- 192.168.12.51 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.139/0.139/0.139/0.000 ms

// Request from "pod-b" to Web server IP which has 192.168.12.10 and 8080 port was listening, if accessed, return "Web server Running" message.
sh-4.4# ip netns exec eb9f2dbf-6872-4318-9ed0-d0bdf6da7f72 curl -vs http://192.168.12.10:8080/
:
< HTTP/1.1 200 OK
< Date: Sat, 02 Jan 2021 08:43:42 GMT
< Server: Apache/2.4.37 (Red Hat Enterprise Linux)
:
< 
Web server Running
* Connection #0 to host 192.168.12.10 left intact

// the source IP was recorded 192.168.12.51("pod-b" net1) to the web server access logs.
192.168.12.51 - - ... "GET / HTTP/1.1" 200 11 "-" "curl/7.61.1"
```

The "pod-b" can access to "pod-a" using IP address.
```console
sh-4.4# ip netns exec eb9f2dbf-6872-4318-9ed0-d0bdf6da7f72 curl -vs http://192.168.12.50:8080/
:
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/2.7.5
:
< 
Pod A
* Closing connection 0
sh-4.4#
```
Next macvlan is similar with ipvlan except MAC address assign different one to each sub-interface with parent interface.

## macvlan plug-in for multiple interfaces with eacg unique MAC address on top of a single interface

You can bind and share one physical interface that is associated with multiple macvlan interfaces directly without a bridge.

![multus_macvlan](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__multus_macvlan_flow_figure5.png)

With macvlan, since connectivity are directly bound with the underlay network using sub-interface having each MAC address,
so it's simple to use like traditional network connectivity.
And you can specify a basic configuration directly in YAML using limited IPAMs. Such as dhcp and static, this time I will use static one.

You specify the `type: SimpleMacvlan` instead of `type: Raw` for a basic configuration in YAML, but you can also configure the same one in JSON either. 
Refer [Configuring a macvlan network with basic customizations](https://docs.openshift.com/container-platform/4.6/networking/multiple_networks/configuring-macvlan-basic.html#configuring-macvlan-basic) for more details.

```console
$ oc edit networks.operator.openshift.io cluster
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks: 
  - name: macvlan-pod-a-main
    namespace: multus-test
    type: SimpleMacvlan
    simpleMacvlanConfig:
      master: ens9
      mode: bridge
      ipamConfig:
        type: static
        staticIPAMConfig:
          addresses:
          - address: 192.168.9.78/24
            gateway: 192.168.9.1
  - name: macvlan-pod-b-main
    namespace: multus-test
    type: SimpleMacvlan
    simpleMacvlanConfig:
      master: ens9
      mode: bridge
      ipamConfig:
        type: static
        staticIPAMConfig:
          addresses:
          - address: 192.168.9.79/24
            gateway: 192.168.9.1
    ...snip...
```

`NetworkAttachmentDefinition` defined in the above CR will be added as follows.

```console
$ oc get net-attach-def -n multus-test
NAME           AGE
macvlan-main   3s
```

The two pods will run on the same worker nodes, and the specified ens9 MAC address is as follows. 
```console
// worker1
ens9: 00:1a:4a:16:06:77
```

After specifying `NetworkAttachmentDefinition` to each pod, you can see redeployed pods as follows.
```console
// Check the pods IP addresses after specifying network you configured.
$ oc patch deploy/pod-a \
  --patch '{"spec":{"template":{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"macvlan-main"}}}}}'
$ oc patch deploy/pod-b \
  --patch '{"spec":{"template":{"metadata":{"annotations":{"k8s.v1.cni.cncf.io/networks":"macvlan-main"}}}}}'

$  oc get pod -o wide -n multus-test
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                         NOMINATED NODE   READINESS GATES
pod-a-77657b97c9-88rx6   1/1     Running   0          23s   10.128.2.19   worker1.ocp46rt.priv.local   <none>           <none>
pod-b-7b66b495b8-98khk   1/1     Running   0          25s   10.128.2.18   worker1.ocp46rt.priv.local   <none>           <none>
```

