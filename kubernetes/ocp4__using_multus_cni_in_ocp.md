# Using the Multus CNI in OpenShift


In this article, I will walk you through a simple summary of Multus and demonstrate how it works in OpenShift using some practical examples. It may be helpful to deepen your understanding of the Multus CNI alongside the official documentation.

## What is the Multus CNI ?
CNI is the container network interface that provides a pluggable application programming interface to configure network interfaces in Linux containers. Multus CNI is such a plugin, and is referred to as a meta-plugin: a CNI plugin that can run other CNI plugins. It works like a wrapper which calls other CNI plugins for attaching multiple network interfaces to pods in OpenShift (Kubernetes). The following terms will be used in this article in order to distinguish them from each other. Refer to the CNI Documentation for more details for each plugin.

Name | Description | Plugins
-----|-------------|--------
Main plugin | Responsible for inserting a network interface into the container network namespace.| bridge, host-device, ipvlan, macvlan and so on.
IPAM plugin | Responsible for managing IP allocation within the Main plugin. | dhcp, host-local, static, whereabouts and so on.
Other plugin | Typically does not work on the primary interface directly. The attributes are set on those interfaces. | tuning, route-override and so on.

Multiple plugins can be configured for a single container using Network Configuration Lists, and an example is shown in the previously referenced document.


## How does Multus work ?
Additional network configurations based on Multus is managed by the Cluster Network Operator (CNO) in OpenShift and the Multus CNI is implemented through the NetworkAttachmentDefinition CR (CustomResource) being led by the Network Plumbing Working Group. Reconciliation is facilitated according to .spec.additionalNetworks in the networks.operator.openshift.io CR. The following diagram describes this process in detail.



Even though you can also directly create NetworkAttachmentDefinition CR without CNO, it is recommended that you use the CNO for the centralized management of your additional networks.

## Main plugins with IPAM
The Main CNI plugin creates and configures network interfaces for pods. I will demonstrate bridge, host-device, ipvlan and macvlan plugins here in that order. Basically, IPAM CNI plugin can be combined with any main plugin, but it does not mean it works well on any environment. First, you should consider the plugins that are appropriate for your network environments before implementing. For example, dhcp IPAM requires an existing DHCP server to be present and available in the network.

IPAM plugin name | Description
-----------------|------------
host-local | IP allocation management within specified ranges on a single node host only. Storage of IP address allocation is specified in flat files local to each host, hence, host-local.
dhcp | IP allocated by an external DHCP server.
whereabouts | IP allocation management within specified ranges across an OpenShift cluster. Storage of IP address allocation is specified in Kubernetes Custom Resources to enable the allocation of an IP across any host in a cluster.
static | IP allocated statically.

The diagrams provided in each of the following sections can be used to aid in the understanding of each plugin feature. They have been intentionally simplified in order to provide a high level overview of the functionality. The tests have been conducted on a OpenShift v4.6.9 (Kubernetes v1.19).

### Bridge
Act as a network switch between multiple pods on the same node host. In its current form, a bridge interface is created which does not link any physical host interface. As a result, connections are not made to any external networks including other pods on the other host nodes.


Configure the bridge plugin with host-local IPAM. The default bridge name is cni0 by default if the name is not specified  using bridge parameter.
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
      "type": "bridge",
      "bridge": "mybr0",
      "ipam": {
          "type": "host-local",
          "subnet": "192.168.12.0/24",
          "rangeStart": "192.168.12.10",
          "rangeEnd": "192.168.12.200"
      }
    }'
    ...snip...
```

Verify the created NetworkAttachmentDefinition, and set it up to the test pods.
```console
$ oc get net-attach-def -n multus-test
NAME          AGE
bridge-main   10s

$ oc patch deploy/pod-a -n multus-test --patch '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
      	    "k8s.v1.cni.cncf.io/networks": "bridge-main"
    	  }
      }
    }
  }
}'
```

List the test pods by checking the associated hosts where they are running and the allocated IP.
```console
$ oc get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP          NODE
pod-a-54b5d785b8-ptzqm   1/1     Running   0          4m12s   10.128.2.2  worker1.ocp4-int.example.com                 pod-a-54b5d785b8-5vfq9   1/1     Running   0          4m12s   10.128.2.13 worker1.ocp4-int.example.com
```

Verify the bridge is not linked to any physical interface and check each pod netns (network namespace) on the scheduled host. As you can see, the connection interface is its own bridge interface, so it cannot access external networks by default. Prompt name in the table means where the command is executed at, you can use ssh, “oc debug node” and “oc rsh” here.

Prompt name | Description
------------|------------
$ | A client host to operate the oc CLI.
worker1 ~# | A compute node host on where the test pod is run on. You can access oc debug node or ssh.
web server ~# | An external host from an OpenShift cluster, httpd web server is running on it.


```console
worker1 ~# ip link show type bridge
29: mybr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
	link/ether 6a:28:59:d0:f2:49 brd ff:ff:ff:ff:ff:ff

worker1 ~# nmcli con show mybr0 | grep connection.interface-name
connection.interface-name:          	mybr0

worker1 ~# ip link show master mybr0
4: vethad403f91@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mybr0 state UP ...
	link/ether 82:c1:92:e1:0d:58 brd ff:ff:ff:ff:ff:ff link-netns NETWORK_NAMESAPCE_1ST_POD-A
30: vethc611bc61@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mybr0 state UP ...
	link/ether 6a:28:59:d0:f2:49 brd ff:ff:ff:ff:ff:ff link-netns NETWORK_NAMESAPCE_2ND_POD-A
```
It’s available to access each other within the test pods on the same host.
```console
worker1 ~# ip netns exec NETWORK_NAMESAPCE_1ST_POD-A ip a | \
           grep -E '10.128|192.168'
	inet 10.128.2.2/23 brd 10.128.3.255 scope global eth0
	inet 192.168.12.10/24 brd 192.168.12.255 scope global net1

worker1 ~# ip netns exec NETWORK_NAMESAPCE_2ND_POD-A ip a | \
           grep -E '10.128|192.168'
	inet 10.128.2.13/23 brd 10.128.3.255 scope global eth0
	inet 192.168.12.11/24 brd 192.168.12.255 scope global net1

worker1 ~# ip netns exec NETWORK_NAMESAPCE_1ST_POD-A ping -c1 -w1 192.168.12.11
PING 192.168.12.11 (192.168.12.11) 56(84) bytes of data.
64 bytes from 192.168.12.11: icmp_seq=1 ttl=64 time=0.219 ms
 
--- 192.168.12.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.219/0.219/0.219/0.000 ms
```
### Host-device
This plugin makes a physical host interface move to a pod network namespace. When enabled, the specified host interface disappears in the root network namespace (default host network namespace). This behavior might affect recreating the pod in place on the same host as the host interface may not be found as it is specified by host-device plugin configuration.



This time, dhcp IPAM is configured and it would trigger the creation of the dhcp-daemon daemonset pods in openshift-multus namespace. The pod in daemon mode listens for an address from a DHCP server on OpenShift, whereas the DHCP server itself is not provided. In other words, it requires an existing DHCP server in the same network. This demonstration shows you the MAC address of the parent is kept in the pod network namespace. Additionally, the source IP and MAC address can be identified by using an external web server access test. 

Add the following configurations.
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
      "device": "ens4",
      "ipam": {
          "type": "dhcp"
      }
    }'
    ...snip...
```

First, check the MAC address of the parent interface on the node host.
```console
worker1 ~# ip link show ens4
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT ...
	link/ether 00:1a:4a:16:06:ba brd ff:ff:ff:ff:ff:ff
```

After setting up host-device-main on test pod-a using k8s.v1.cni.cncf.io/networks annotation like the previous bridge section,  check whether the host interface is shown as follows.
```console
$ oc get pod -o wide
NAME                   READY STATUS   RESTARTS  AGE  IP           NODE
Pod-a-55f5cf54cb-428qs 1/1   Running  0         56s  10.128.2.17  worker1.ocp4-int.example.com

worker1 ~#  ip link show ens4
Device "ens4" does not exist.
```

You can also use POD ID to find out your netns (network namespace) on the host as well as checking if the MAC address is the same with the physical interface on the host using the netns. If the netns is not displayed as described in the following section, it means you will need to find the netns implicitly.
```console
worker1 ~# crictl pods
POD ID  CREATED       STATE  NAME                    NAMESPACE                            	
POD_ID  6 minutes ago Ready  pod-a-55f5cf54cb-428qs  multus-test                          	

worker1 ~# crictl inspectp  --output json POD_ID | \
           jq '.info.runtimeSpec.linux.namespaces | .[] | select(.type == "network")'
{
  "type": "network",
  "path": "/var/run/netns/NETWORK_NAMESPACE_OF_THE_POD"
}

worker1 ~# ip netns exec NETWORK_NAMESPACE_OF_THE_POD ip a | \
           grep -v -E 'lo:|127\.0|valid_lft|inet6'
3: eth0@if27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default  
	link/ether 0a:58:0a:80:02:11 brd ff:ff:ff:ff:ff:ff link-netnsid 0
	inet 10.128.2.17/23 brd 10.128.3.255 scope global eth0
4: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
	link/ether 00:1a:4a:16:06:ba brd ff:ff:ff:ff:ff:ff
	inet 192.168.12.101/24 brd 192.168.12.255 scope global net1
```

You can test connectability between an external web server and a test pod. At that point, the host interface MAC address was used.
```console
web server ~# curl http://192.168.12.101:8080/
Pod A

worker1 ~# ip netns exec NETWORK_NAMESPACE_OF_THE_POD \
           curl http://192.168.12.10:8080/
Running Web server

web server ~# tail -n1 /var/log/httpd/access_log
192.168.12.101 - - [12/Jan/2021:16:38:50 +0900] "GET / HTTP/1.1" 200 19 "-" "curl/7.61.1"

web server ~# ip neigh | grep 192.168.12.101
192.168.12.101 dev ens9 lladdr 00:1a:4a:16:06:ba STALE
```

This behavior is similar to the macvlan passthru mode, but passthru mode works as a sub-interface, so the parent interface does not move to a pod network namespace.
### Ipvlan
The ipvlan plugin may be used in the event that there are limited MAC addresses that can be used. This issue is common on some switch devices which restrict the maximum number of MAC addresses per physical port due to port security configurations. When operating in most cloud providers, you should  consider using ipvlan instead of macvlan as unknown MAC addresses are forbidden in VPC networks.

The sub-interface of the ipvlan can use distinct IP addresses with the same MAC address of the parent host interface. So, it would not work well with a DHCP server which depends on the MAC addresses. Parent host interface acts as a bridge (switch) with L2 mode, and it acts as a router with L3 mode of the ipvlan plug-in. In the following example, I use the whereabouts IPAM and L2 mode(default) will be used. Refer to the IPVLAN Driver HOWTO for additional details.
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
  	"master": "ens4",
  	  "ipam": {
      	    "type": "whereabouts",
      	    "range": "192.168.12.0/24",
      	    "range_start": "192.168.12.50",
      	    "range_end": "192.168.12.200"
  	  }
	}'
	...snip…
```

With ipvlan and whereabouts IPAM, it’s easy to test connectivity among pods and an external web server without external system dependencies. The ipvlan sub-interfaces work like the linux IP alias.

You can list each ipvlan and the associated parent MAC address in the pod using the “oc netns” command.
```console
$ oc get pod -o wide -n multus-test
NAME                   READY  STATUS  RESTARTS AGE IP          NODE
Pod-a-7dc6dd58d9-p4xk7 1/1    Running 0        4m  10.128.2.13 worker1.ocp4-int.example.com
pod-a-7dc6dd58d9-knrqk 1/1    Running 0        4m  10.128.2.15 worker1.ocp4-int.example.com

worker1 ~# ip link show ens4
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT ...
	link/ether 00:1a:4a:16:06:ba brd ff:ff:ff:ff:ff:ff

worker1 ~# ip netns exec NETWORK_NAMESAPCE_1ST_POD-A ip link show type ipvlan
4: net1@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN ...
	link/ether 00:1a:4a:16:06:ba brd ff:ff:ff:ff:ff:ff

worker1 ~# ip netns exec NETWORK_NAMESAPCE_2ND_POD-A ip link show type ipvlan
4: net1@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN ...
	link/ether 00:1a:4a:16:06:ba brd ff:ff:ff:ff:ff:ff
```

The test pods can access each other and an external web server as follows.
```console
worker1 ~# ip netns exec NETWORK_NAMESAPCE_1ST_POD-A ip a | grep -E '10.128|192.168'
	inet 10.128.2.13/23 brd 10.128.3.255 scope global eth0
	inet 192.168.12.51/24 brd 192.168.12.255 scope global net1

worker1 ~# ip netns exec NETWORK_NAMESAPCE_2ND_POD-A ip a | grep -E '10.128|192.168'
	inet 10.128.2.15/23 brd 10.128.3.255 scope global eth0
	inet 192.168.12.52/24 brd 192.168.12.255 scope global net1

worker1 ~# ip netns exec NETWORK_NAMESAPCE_2ND_POD-A ping -c1 -w1 192.168.12.51
PING 192.168.12.51 (192.168.12.51) 56(84) bytes of data.
64 bytes from 192.168.12.51: icmp_seq=1 ttl=64 time=0.079 ms
 
--- 192.168.12.51 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.079/0.079/0.079/0.000 ms

worker1 ~# ip netns exec NETWORK_NAMESAPCE_2ND_POD-A curl http://192.168.12.1:8080/
Running Web server

web server ~# tail -n1 /var/log/httpd/access_log
192.168.12.52 - - [12/Jan/2021:17:56:52 +0900] "GET / HTTP/1.1" 200 19 "-" "curl/7.61.1"

web server ~# curl http://192.168.12.52:8080/
Pod A
```
### Macvlan
With macvlan, since the connectivity is directly bound with the underlying network using sub-interfaces each having MAC address, so it's simple to use as it aligns to traditional network connectivity.

At the moment, only macvlan supports configuration in both formats: SimpleMacvlan(YAML) and Raw(JSON). Further YAML format information is available in the Configuring a macvlan network with basic customizations reference. Currently, YAML format provides limited configuration, so the macvlan with static IPAM as JSON format will be demonstrated.
macvlan generates MAC addresses per the sub-interfaces, and in most cases, Public Cloud Platforms are not allowed due to their security policy and, Hypervisors have limited capabilities. For the RHV (Red Hat Virtualization) use case, you will need to change “No network filter” on your network profile before executing the test. For vSwitch in vSphere environments, similar relaxed policies need to be applied.

The test procedure is almost the same as ipvlan, so it is easy to compare both plugins.

Macvlan has multiple modes. In this example, bridge mode will be configured. Refer to MACVLAN documentation for more information on the other mode which will not be demonstrated.
```console
$ oc edit networks.operator.openshift.io cluster
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks:
  - name: macvlan-main
    namespace: multus-test
    type: Raw
    rawCNIConfig: '{
       "cniVersion": "0.3.1",
  	"name": "macvlan-main",
  	"type": "macvlan",
  	"mode": "bridge",
  	"master": "ens4",
  	"ipam": {
    	    "type": "static"
  	}
    }'
    ...snip...
```

Static IPAM can specify the IP address dynamically using the “ips” capabilities argument in each pod as follows. Refer to the Supported arguments reference in the static IPAM for more details. In this example, two test pods are specified with 192.168.12.77/24 address for pod-a, and 192.168.12.78/24 address for pod-b.
```console
$ oc edit deploy pod-a -n multus-test
apiVersion: apps/v1
kind: Deployment
:
spec:
  :
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: '[
          {
            "name": "macvlan-main",
	     "ips": [
  	       "192.168.12.77/24"
	     ]
          }
        ]'
       ...snip...
```

You can list each macvlan which has each MAC address in the pod using the “oc netns” command. There are differences from the previous ipvlan example. Run two test pods to demonstrate this functionality.
```console
$ oc get pod -o wide -n multus-test
NAME                   READY STATUS  RESTARTS AGE IP           NODE
pod-a-858778d89f-q5l7x 1/1   Running 0        46s 10.128.3.52  worker1.ocp4-int.example.com
pod-b-64c9ddb9fd-5pmd8 1/1   Running 0        61s 10.128.2.164 worker1.ocp4-int.example.com
```

First, check the parent interface MAC address for comparison of the macvlan sub-interfaces.
```console
worker1 ~# ip link show ens4
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
	link/ether 00:1a:4a:16:06:ba brd ff:ff:ff:ff:ff:ff
```

Next, check the MAC address of the two test pods. As you can see, the MAC addresses are different from one another.
```console
worker1 ~# ip netns exec NETWORK_NAMESPACE_FOR_POD-A ip link show type macvlan
4: net1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default  
	link/ether ca:15:c7:02:61:53 brd ff:ff:ff:ff:ff:ff link-netnsid 0

worker1 ~# ip netns exec NETWORK_NAMESPACE_FOR_POD-B ip link show type macvlan
4: net1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default  
	link/ether f6:e6:29:ff:76:0e brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

The test pods can access each other and an external web server as follows. As you can see, the ARP record for pod-b shows you the different MAC address with the associated parent interface on the node host.
```console
worker1 ~# ip netns exec NETWORK_NAMESPACE_FOR_POD-A ip a | grep -E '10.128|192.168'
	inet 10.128.3.52/23 brd 10.128.3.255 scope global eth0
	inet 192.168.12.77/24 brd 192.168.12.255 scope global net1
worker1 ~# ip netns exec NETWORK_NAMESPACE_FOR_POD-B ip a | grep -E '10.128|192.168'
	inet 10.128.2.170/23 brd 10.128.3.255 scope global eth0
	inet 192.168.12.78/24 brd 192.168.12.255 scope global net1

worker1 ~# ip netns exec NETWORK_NAMESPACE_FOR_POD-B ping -c1 -w1 192.168.12.77
PING 192.168.12.77 (192.168.12.77) 56(84) bytes of data.
64 bytes from 192.168.12.77: icmp_seq=1 ttl=64 time=0.081 ms
 
--- 192.168.12.77 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.081/0.081/0.081/0.000 ms

worker2 ~# ip netns exec NETWORK_NAMESPACE_FOR_POD-B curl http://192.168.12.1:8080/
Running Web server

web server ~# tail -n1 /var/log/httpd/access_log
192.168.12.78 - - [17/Jan/2021:16:57:08 +0900] "GET / HTTP/1.1" 200 19 "-" "curl/7.61.1"

web server ~# curl http://192.168.12.78:8080/
Pod B
web server ~# ip neigh | grep 192.168.12.78
192.168.12.78 dev ens9 lladdr f6:e6:29:ff:76:0e REACHABLE
```

Up to this point, we demonstrated the main and IPAM plugins. In most cases, you may configure additional networks using one of the CNI plugin combinations that has been previously covered, reviewed. However, you might need to customize the base network configuration depending on your specific use case or environment dependencies. If this is indeed the case, Other plugin features may be helpful.

Other plugins
These plugins usually set attributes of interfaces not handling the primary interface directly. As stated previously, the specific configurations depend on the network topology and hardware. In this section,  the tuning and route-override plugins will be demonstrated.

Plugin name | Description
------------|------------
tuning | This plugin can change sysctls belonging to the network namespace and several interface attributes (MTU, MAC address and so on).
route-override | This plugin can modify IP routes in the network namespace.

Further information is provided here: tuning plugin and route-override-cni provide additional details.

Multiple standard CNI plugins can be specified in the “plugins” section as follows.
```console
$ oc edit networks.operator.openshift.io cluster
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks:
  - name: tuning-and-route-override-meta
    namespace: multus-test
    type: Raw
    rawCNIConfig: '{
      "cniVersion": "0.3.1",
      "name": "tuning-and-route-override-meta",
      "plugins": [
        {
          "type": "macvlan",
          "mode": "bridge",
          "master": "ens4",
          "ipam": {
            "type": "static"
          }
        },
        {
          "type": "tuning",
          "sysctl": {
            "net.core.somaxconn": "512"
          }
        },
        {
          "type": "route-override",
          "addroutes": [
            {
              "dst": "192.168.9.0/24",
              "gw": "192.168.12.1"
            }
          ]
        }
      ]
    }'
    ...snip...
```

The changes can be verified in the netns of the test pod as follows.
```console
worker ~# ip netns exec NETWORK_NAMESPACE_OF_THE_POD cat /proc/sys/net/core/somaxconn
512

worker ~# ip netns exec NETWORK_NAMESPACE_OF_THE_POD route -n
Kernel IP routing table
Destination 	Gateway     	Genmask     	Flags Metric Ref	Use Iface
0.0.0.0     	10.128.2.1  	0.0.0.0     	UG	0  	0    	0 eth0
:
172.30.0.0  	10.128.2.1  	255.255.0.0 	UG	0  	0    	0 eth0
192.168.9.0 	192.168.12.1	255.255.255.0   UG	0  	0    	0 net1
192.168.12.0	0.0.0.0     	255.255.255.0   U 	0  	0    	0 net1
```
## Conclusion
Throughout this article, a variety of use cases were demonstrated by attaching to additional networks using CNI plugins along with Multus CNI on OpenShift. While focus was placed on each plugin type, they can work together and even be chained with one another. As a result, a more complex and powerful set of networking constructs can be produced.  While it may be difficult to understand each of the concepts initially, it is the hope that through this post, you were able to capture the primary benefits and use it to further deepen your knowledge on the topic.

Thank you for reading.
