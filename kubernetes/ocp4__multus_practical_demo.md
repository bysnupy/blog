# OpenShift and Multus Practical Dive

## Summary

This article is a practical tutorial that demonstrates how to configure Multus on the running OpenShift 4.6 cluster.
OpenShift Container Platform uses the Multus CNI for providing additional networks.
It allows to define an additional network based on the various plugins of the Multus and attach one or more of these networks to your pods.

Basically, Multus does not setup network directly, just call other various CNI plugins based on the Multus definition.
Refer [Demystifying Multus](https://www.openshift.com/blog/demystifying-multus) for more details.
Sometimes, it's confusing how to configure and work each other among multiple CNI plugins defined in the Multus.
So I'd like to help your more understanding it through multiple practical demonstrations here.

## Manifest overview

OpenShift manages the additional network definition of the Multus using the Cluster Network Operator(CNO) and its CustomerResource(CR).
When you specify an additional network to create, the CNO creates the `NetworkAttachmentDefinition` object automatically,
you can also your own `NetworkAttachmentDefinition`. 
But you do not edit the `NetworkAttachmentDefinition` that CNO manages, it may disrupt your additional network.

Let's see how to construct the CR for defining additional network as in the following example. 
I will introduce limited plugins, such as bridge, host-device, ipvlan, macvlan within main plugins due to my limited test resources.

```yaml
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalNetworks:
  - name: YOUR-NET-ATTACH-DEF-NAME
    namespace: TARGET-NAMESPACE
    type: Raw
    rawCNIConfig: |
    '{
        "cniVersion": "0.3.1",
        "name": "YOUR-NET-ATTACH-DEF-NAME",
        "type": "bridge|host-device|ipvlan|macvlan",
        :
        Network Interface creating Main plugin section
        :
        "ipam": {
            "type": "static|dhcp|host-local|whereabouts"
            :
            Network setting IPAM plugin section
            :
        }
    }'
```
