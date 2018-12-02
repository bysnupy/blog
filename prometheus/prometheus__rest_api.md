# REST API in Prometheus

## Summary

How to get the `metrics` using `REST API` of `Prometheus`.

## Example

A simple usage using `curl` command as follows.

~~~
# curl -ks -H 'Authorization: Bearer <TOKEN>' \
     https://prometheus-k8s-openshift-monitoring.router.default.svc.cluster.local/api/v1/query?query=${QUERY_EXPRESSION}
~~~

- Metrics and Query

Metrics | Query | Unit
-|-|-
all node cpu usgae | node:node_cpu_utilisation:avg1m | Percentage(%)
all node memory usage | node:node_memory_utilisation | Percantage(%)
pod cpu usage | all namespace_pod_name_container_name:container_cpu_usage_seconds_total:sum_rate | Seconds(s)
container_memory_usage_bytes | all container memory usage | Bytes
project cpu usage | sum(namespace_pod_name_container_name:container_cpu_usage_seconds_total:sum_rate{namespace="your project"}) | Senconds(s)
project memory usage | sum(container_memory_usage_bytes{namespace="your project", container_name!=""}) | Bytes

