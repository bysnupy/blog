# How does Multus work on OpenShift ?

## Summary

OpenShift v4 is an Operator-Managed platform and Multus CNI is also managed by the Cluster Network Operator(CNO).
We will discuss how you should use it on your OpenShift cluster for your multiple networks.

## Using Multus on OpenShift

OpenShift allows a pod to have multiple interfaces using Multus CNI that chains multiple plugins for the additional network attachment and management through the `NetworkAttachmentDefinition` object. In OpenShift case, the Cluster Network Operator(CNO) manages the `NetworkAttachmentDefinition` using CustomResource(CR).
When you specify an additional network to the CNO CR, the CNO generates the `NetworkAttachmentDefinition` object automatically according to the CNO CR.

Note that there are multiple network interface and IPAM(IP address management) options, and we should use network options appropriately for your use case.

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
