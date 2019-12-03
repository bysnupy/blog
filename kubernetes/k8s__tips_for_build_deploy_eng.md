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
-------------------------------|--------------------------------------
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
 
worker-0 ~# cat \
/sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podYYYYYYYY_YYYY_YYYY_YYYY_YYYYYYYYYYYY.slice/crio-XXX... Container ID ...XXX.scope/cpu.shares
32
```

Additionally, you should add `kube-reserved` and `system-reserved` to provide more reliable scheduling and minimize node resource overcommitment in all nodes.
Refer [Allocating resources for nodes in an OpenShift Container Platform cluster](https://docs.openshift.com/container-platform/4.2/nodes/nodes/nodes-nodes-resources-configuring.html) for more details.

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: set-allocatable
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-kubelet: small-pods
  kubeletConfig:
    systemReserved:
      cpu: 500m
      memory: 512Mi
    kubeReserved:
      cpu: 500m
      memory: 512Mi
```

If you have more than 1000 nodes, you can consider to tune [percentageOfNodesToScore](https://kubernetes.io/docs/concepts/scheduling/scheduler-perf-tuning/) for scheduler performance tuning, 
this feature state is beta as of Kubernetes 1.14(OpenShift 4.2).

## Image Registry

This component work on image push and pull tasks by build and deployment.
First of all, you should consider to adopt appropriate storage and network enough to
process required traffic and IO for maximum concurrent image pull and push you assumed.
Usually object storage is recommended,
because it is atomic, meaning the data is either written completely or not written at all, even if there is a failure during the write.
And it can share a volume with other duplicated registry as RWX(ReadWriteMany) mode.
Further information is here:[Recommended configurable storage technology](https://docs.openshift.com/container-platform/4.2/scalability_and_performance/optimizing-storage.html#recommended-configurable-storage-technology_persistent-storage).
You can also consider `ImagePullPolicy` to reduce pull/push workload as follows.

* [Image pull policy](https://docs.openshift.com/container-platform/4.2/openshift_images/managing-images/image-pull-policy.html)

Value | Description
-|-
Always | Always pull the image.
IfNotPresent | Only pull the image if it does not already exist on the node.
Never | Never pull the image.

## Build Pod

When build Pod is created and start to build application, all control passes to the build Pod,
and work through build configurations are defined by a `BuildConfig`.
Depending on how you choose to create your application and how to provide source content to build and operate on affect build performance.
For instance, if you build Java or nodejs application using `maven` or `npm`, you may download many libraries from their external repositories.
At that time the repositories or access path have some performance issue, then the build process will be failed or delayed more than usual time.
It means if your build depend on external service or resource, it's easy to affect from that regardless of your system status.
So you can consider to create local repository (maven, npm, git and so on) for reliable and stable performance for build.
Or you can reuse previously downloaded dependencies, previously built artifacts as using Source-to-Image (S2I) incremental builds, 
if your image registry performance is enough to pull previously-built images for every incremental builds.

* [Performing Source-to-Image (S2I) incremental builds](https://docs.openshift.com/container-platform/4.2/builds/build-strategies.html#builds-strategy-s2i-incremental-builds_build-strategies)
```yaml
strategy:
  sourceStrategy:
    from:
      kind: "ImageStreamTag"
      name: "incremental-image:latest" 
    incremental: true
```

**TODO: docker image layerd cache**

You should check build logs for troubleshooting in build process, because Kubernetes can only detect build Pod is complete successfully or not.

## Application Pod

Like the Build Pod, all controls passes to the application Pod, when application Pod is created, and work through application implementation.
If the application depends on external services and resources to initialize, such as DB connection pooling, KVS and other API connections.
For instance, if DB server that is required to connection pooling have performance issues or reach the maximum connection count while application Pod is starting, 
the application Pod initialization can delay more than you expected, so if your application have external dependencies, you also check them whether they are running well.
And if `Readiness Probes` and `Liveness Probes` is configured for your application Pod, you should set `initialDelaySeconds` and `periodSeconds` enough to your application Pod initialized.
If it's too short `initialDeplaySeonds` and `periodSeconds` to check the application states, your application will be restarted uselessly, it result in delay to deploy application Pod.

* [Monitoring container health](https://docs.openshift.com/container-platform/4.2/nodes/containers/nodes-containers-health.html)
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness-http
    image: k8s.gcr.io/liveness 
    args:
    - /server
    livenessProbe: 
      httpGet:   
        # host: my-host
        # scheme: HTTPS
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 15  
      timeoutSeconds: 1   
    name: liveness
```

## Build and Deployment in parallel

Lastly, I recommend to keep concurrent build and deployment tasks in one cycle as appropriate numbers for suppressing resource issues. 
As you seen here, these tasks are chained with other tasks and it will iterate automatically through their lifecycles, 
so it would happen more workload than you expected. Usually compile(build) and application initialization(deployment) process is a CPU aggressive task, 
you may need to evaluate what count concurrent build and deployment tasks is available and stable on your cluster before scheduling their tasks.

## Conclusion

We took a look how each part affect each other within build and deployment processes.
You may already know each configuration I introduced above and it's nothing special.
As I introduced the configuration and information set for only build and deployment performance topic, 
I'd like to help your understanding about interaction on the build and deployment here. 
I hope it's helpful for your stable system management. Thank you for reading this article.
