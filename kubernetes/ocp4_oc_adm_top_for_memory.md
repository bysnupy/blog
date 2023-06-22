# Using "oc adm top" for monitoring memory usage

To provide your reliable services, you need to monitor your services performance by examining the containers and pods in the OpenShift(Kubernetes) clusters. The OpenShift provides "oc adm top" command for that purpose. It depends on "kubectl adm top" internally, and basically it's the same function with it.

Among others, I'd like to show you the details of the command aspect of monitoring memory resources this time. The memory metrics in OpenShift would be a bit different with the legacy linux one provided from "top" and "free". So I'd like to help your understanding around memory resources in OpenShift for suppressing confusion around this through this article.

## Using "oc adm top pods" for monitoring memory usage of pods

The "oc adm top pods" shows us the memory usage of pods based on "container_memory_working_set_bytes" metrics as follows through "prometheus-adapter" service in "openshift-monitoring" which is provided according to the metrics rules in the prometheus-adapter configuration.



```console
// The memory metrics depends on "v1beta1.metrics.k8s.io" API service,
// and it's provided by "prometheus-adapter" according to the metrics rules, container_memory_working_set_bytes.
$ oc adm top pod -n test
NAME                     CPU(cores)   MEMORY(bytes)   
httpd-675fd5bfdd-96br8   1m           53Mi        

$ oc get pods.metrics.k8s.io httpd-675fd5bfdd-96br8 -n test -o yaml
apiVersion: metrics.k8s.io/v1beta1
containers:
- name: httpd
  usage:
    cpu: 1m
    memory: 55048Ki
kind: PodMetrics
:

$ oc get apiservice v1beta1.metrics.k8s.io
NAME                     SERVICE                                   AVAILABLE   AGE
v1beta1.metrics.k8s.io   openshift-monitoring/prometheus-adapter   True        21d

$ oc get cm/adapter-config -o yaml -n openshift-monitoring
apiVersion: v1
data:
  config.yaml: |-
    "resourceRules":
:
      "memory":
        "containerLabel": "container"
        "containerQuery": |
          sum by (<<.GroupBy>>) (
            container_memory_working_set_bytes{<<.LabelMatchers>>,container!="",pod!=""}
          )
:
```

And the "container_memory_working_set_bytes" is collected from the cgroup memory control files as follows.

  e.g.> Refer Source codes about the workingset for more details.
  container_memory_working_set_bytes = memory.usage_in_bytes - total_inactive_file

The "top" command in linux shows process memory usage using each column of VIRT(virtual memory), RES(actual memory), SHR(shared memory). It's a different point with OpenShift pods memory usage values, and we need to check why OpenShift(Kubernetes) adapted the working set.
In k8s docs, Measuring resource usage - Memory, the reason is, the "working set" is the amount of memory in-use that cannot be freed under memory pressure.

```text
In an ideal world, the "working set" is the amount of memory in-use that cannot be freed under memory pressure. However, calculation of the working set varies by host OS, and generally makes heavy use of heuristics to produce an estimate.

The Kubernetes model for a container's working set expects that the container runtime counts anonymous memory associated with the container in question. The working set metric typically also includes some cached (file-backed) memory, because the host OS cannot always reclaim pages.
```

In other words, we can say that "working set" is appropriate metrics for monitoring OOM limitation if we set up resources.limits.memory limitation in pods. As you know, when reaching out the memory limits size, the “cgroup” cuts them out with OOM events, if it's not successful after trying to reclaim memory such as some kinds of caches.

However, the “working set” includes some kind of cache such as the page cache which is for disk io performance actually. For example, if loading big size files in processes, then we need to consider that an unexpected page cache would be bigger and it makes the working set be bigger than actual in-use memory size not reclaimed for the pod.

As a result, the "working set" intends to show us usually how much memory is required to run the pods, but even though the "working set" is bigger than the limits size, it does not always trigger OOM events due to the caches. So we need to consider that during monitoring pods memory, the "working set" just provides us estimate memory usage and trends.

## Using "oc adm top nodes" for monitoring memory usage of nodes

The "oc adm top nodes" shows us a bit different memory usage compared to "oc adm top pods", it means the memory usage is for evaluation whether it's enough to scheduling pods to the nodes.
By default, "oc adm top nodes" is based on "Allocatable" logical total size of the memory which is not actual total size. If you want to check the actual memory usage of your nodes, you need to run "oc adm top nodes --show-capacity" instead.

Please remember, OpenShift(Kubernetes) is a kind of container platform, and the most important workload unit is a Pod, so we should recognize that memory resources are different with the legacy linux hosts.
If you try to apply the same resource mindset of Linux hosts to OpenShift(Kubernetes) nodes, it leads to misunderstanding as a result. The "Allocatable" is a total resource size for those pods, the k8s scheduler just only considers the memory is enough to schedule the pods based on "Node Allocatable".

The memory total size of "Allocatable" is calculated as follows.

```[Allocatable] = [Node Capacity] - [system-reserved] - [Hard-Eviction-Thresholds] - [Pre-allocated memory like huge pages]```

And the node memory usage metrics is collected as "node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes". As a result, "oc adm top nodes" shows us according to the "(( the node memory usage ) / ( Allocatable ) x 100)" formula simply.

```console
// "--show-capacity" shows us the node memory usage
// based on "MemTotal" in "/proc/meminfo" instead of "Allocatable".

$ oc adm top nodes dapark-osp-2jpnc-worker-0-v5mpd
NAME                            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
test-ocp-2jpnc-worker-0-v5mpd   952m         12%    7986Mi          53%       

$ oc adm top nodes dapark-osp-2jpnc-worker-0-v5mpd --show-capacity
NAME                            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
test-ocp-2jpnc-worker-0-v5mpd   638m         7%     8041Mi          50%       

// As follows, the memory usage of nodes is the same with,
// "node_memory_MemAvailable_bytes" is deducted from "node_memory_MemTotal_bytes".
$ oc get cm/adapter-config -o yaml -n openshift-monitoring
apiVersion: v1
data:
  config.yaml: |-
    "resourceRules":
:
"memory":
  :
  "nodeQuery": |
    sum by (<<.GroupBy>>) (
      node_memory_MemTotal_bytes{job="node-exporter",<<.LabelMatchers>>}
      -
      node_memory_MemAvailable_bytes{job="node-exporter",<<.LabelMatchers>>}
    )
:
```

In nature, the "Allocatable" is not an actual total size, and it's just calculated simply from the above formula.
For example, the "MEMORY%" column value can be bigger than 100% when "Allocatable" is deducted than the actual usages. Specifically, if you set many huge pages up on your nodes, even though the huge pages were deducted from "Allocatable" total in advance, the node memory usage includes all the huge pages as a kind of pre-allocated memory. As a result, memory usage doubled the huge pages size during the calculation, and the discrepancy is bigger than our expectation. But currently, it's a design of "oc adm top nodes" which does not revise the metrics and keeps simple implementation.

Except for the huge pages case, the discrepancy is not so big in usual. And you consider not only "oc adm top nodes", but also "oc adm top nodes --show-capacity" for your better capacity planning of pods.

## Conclusion

While focus was placed on monitoring pods and nodes memory usage, "oc adm top" would be useful for a variety of use cases according to your needs. I hope you use it to further deepen your knowledge of monitoring in OpenShift(Kubernetes) through this article.

Thank you for reading.
