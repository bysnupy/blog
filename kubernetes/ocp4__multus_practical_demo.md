# How does Multus work on OpenShift 4 with Operator ?

## Summary

OpenShift v4 is an Operator-Managed platform and Multus CNI is also managed by the Cluster Network Operator(CNO).
We will discuss how you should use it on your OpenShift cluster for your multiple networks.

## Multus on OpenShift

OpenShift manages the additional network definition of the Multus using the Cluster Network Operator(CNO) and its CustomResource(CR).
When you specify an additional network to the CNO CR, the CNO generates the `NetworkAttachmentDefinition` object automatically according to the CNO CR. 
But you should not manually edit the `NetworkAttachmentDefinition` that CNO manages, it may disrupt your additional network.

Let's see how to construct the CR for defining additional network as in the following figure. 
You can see some details about limited plugins here, such as bridge, host-device, ipvlan, macvlan within main plugins due to my limited test resources. 
But it would be enough to make sense.

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
