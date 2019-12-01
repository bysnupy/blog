# Title: Tips for build and deployment, stable and reliable OpenShift.

In this article, I will introduce some helpful common tips to manage reliably build and deployment on OpenShift.
If you have been experienced sudden performance degradation of build and deployment on running OpenShift cluster,
it may be helpful to troubleshoot your system.
We start to review whole processes from build to deployment, and then cover each process for more details.

## Processes from build to deployment

When CI/CD or user triggers the build for a Pod deployment, the processes are similar with a following image.

![build_deploy_process](https://github.com/bysnupy/blog/blob/master/kubernetes/build_deploy_performance.png)

As you can see, all parts are not controlled over by only Kubernetes,
usually main processes for build and deployment depend on own configuration and external dependencies in the systems.
We can split into each process by what component have responsibility to manage as follows.

Responsibility                 |  Component
---------------------------------------------------------------------
Schedule and Control a Pod     |  Kubernetes
Manage images                  |  Image Registry
Build image for an application |  build Pod
Deploy an application          |  Pod created by deployment work

## Kubernetes

In this process, Kubernetes provides resources based on `buildConfig` and `deploymentConfig`.
Usually build and deploy will be failed before starting their work, if any trouble happen in handling resources,
such as volume mount failure, scheduling pending and so on. And you can also find the reasons easily in the event logs.
So if build and deployment is successful, even though the time took more than before, it would not be Kubernetes scheduling and control problem.
For clarify the performance issue,
we should ensure `buildConfig` and `deploymentConfig` are configured with enough resource requests through the following examples.

* Sample for `buildConfig`,
```yaml
apiVersion: "v1"
kind: "BuildConfig"
metadata:
  name: "sample-build"
spec:
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
```

* Sample for `deploymentConfig`,
```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
spec:
  template:
    spec:
      containers:
      - image: test-image
        name: container1
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
```

You should allocate resource size enough to build and deploy reliably after testing, keep in your mind it's not minimum resource setting.
For instance, if you set "resources.requests.cpu: 32m", it'll assign more CPU time through cpu.shares control group parameter as follows.

```
# oc describe pod test-1-abcde | grep -E 'Node:|Container ID:|cpu:'
Node:               worker-0.ocp.example.local/x.x.x.x
    Container ID:  cri-o://XXX... Container ID ...XXX
      cpu:        32m
# oc get pod test-5-8z5lq -o yaml | grep -E '^  uid:'
  uid: YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY
 
# cat /sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podYYYYYYYY_YYYY_YYYY_YYYY_YYYYYYYYYYYY.slice/crio-XXX... Container ID ...XXX.scope/cpu.shares
32
```

Additionally, you should add `kube-reserved` and `system-reserved` to provide more reliable scheduling and minimize node resource overcommitment in all nodes.

## Image Regisry
