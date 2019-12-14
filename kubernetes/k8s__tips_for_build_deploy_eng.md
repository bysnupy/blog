# How to keep build and deployment stable performance on OpenShift ?

In this article, I will introduce some helpful, common tips to manage reliable builds and deployments on OpenShift. If you have experienced a sudden performance degradation for builds and deployments on OpenShift, it may be helpful to troubleshoot your cluster. We start by reviewing the whole processes from build to deployment, and then cover each aspect in more detail. We will use OpenShift 4.2 (Kubernetes 1.14) for this purpose.

When a CI/CD process or a user triggers the build for a pod deployment, the processes will proceed as shown in the following image, which is simplified for readability.

![build_deploy_process](https://github.com/bysnupy/blog/blob/master/kubernetes/build_deploy_performance.png)

As you can see, Kubernetes does not control all of the processes, but the main processes for build and deployment depend on the user’s application build configuration and dependencies on external systems. We can split out each process by which component has responsibility to manage things as follows.

Responsibility|Component
-|-
Schedule and Control a Pod | Kubernetes
Manage images | Image Registry
Build image for an application | build Pod
Deploy an application|application Pod

## Kubernetes

In this process, Kubernetes provides resources based on the BuildConfig and DeploymentConfig. Usually, a build and deploy will fail before starting their work if there is any trouble handling resources, such as volume mount failure, scheduling pending, and so on. And you can usually find the reasons easily in the event logs.

So if build and deployment is successful, even though the time took more than before, it would not be a Kubernetes scheduling and control problem. For clarifying the performance issue, we should ensure BuildConfig and DeploymentConfig are configured with enough resource requests as in the following examples.

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

You should allocate resources large enough to build and deploy reliably after testing, keeping in mind that it’s not a minimum resource setting.

For instance, if you set resources.requests.cpu: 32m, it will assign more CPU time through the cpu.shares control group parameter as follows.

```command
# oc describe pod test-1-abcde | grep -E 'Node:|Container ID:|cpu:'
Node:               worker-0.ocp.example.local/x.x.x.x
    Container ID:  cri-o://XXX... Container ID ...XXX
      cpu:        32m
# oc get pod test-5-8z5lq -o yaml | grep -E '^  uid:'
  uid: YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY
 
worker-0 ~# cat \
/sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podYYYYYYYY_YYYY_YYYY_YYYY_YYYYYYYYYYYY.slice/crio-XXX... Container ID ...XXX.scope/cpu.shares
32
```
As of OpenShift 4.1, you can manage a cluster wide build defaults including above resource requests through Build resource. Refer Build configuration resources for more details.

```yaml
apiVersion: config.openshift.io/v1
kind: Build
metadata:
  name: cluster
spec:
  buildDefaults:
    defaultProxy:
      httpProxy: http://proxy.com
      httpsProxy: https://proxy.com
      noProxy: internal.com
    env:
    - name: envkey
      value: envvalue
    resources:
      limits:
        cpu: 100m
        memory: 50Mi
      requests:
        cpu: 10m
        memory: 10Mi
  buildOverrides:
    nodeSelector:
      selectorkey: selectorvalue
operator: Exists
```

Additionally, you should add kube-reserved and system-reserved to provide more reliable scheduling and minimize node resource overcommitment in all nodes. Refer to Allocating resources for nodes in an OpenShift Container Platform cluster for more details.

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

If you have more than 1000 nodes, you can consider to tune percentageOfNodesToScore for scheduler performance tuning, this feature state is beta as of Kubernetes 1.14 (OpenShift 4.2).

##Image Registry

This component works on image push and pull tasks for both builds and deployments. First of all, you should consider adopting appropriate storage and network resources to process the required I/O and traffic for the maximum concurrent image pull and push you estimated. Usually object storage is recommended, because it is atomic, meaning the data is either written completely or not written at all, even if there is a failure during the write. And it can also share a volume with other duplicated registry as RWX (ReadWriteMany) mode. Further information can be found in the Recommended configurable storage technology documentation.

You can also consider IfNotPresent ImagePullPolicy to reduce pull/push workload as follows.

* Image pull policy

Value|Description
-|-
Always|Always pull the image.
IfNotPresent|Only pull the image if it does not already exist on the node.
Never|Never pull the image

## Build Pod

When a build pod is created and starts to build an application, all control passes to the build pod, and the work is configured through build configurations parameters defined by a BuildConfig. How you choose to create your application and provide source content to build and operate on all affect build performance.

For instance, if you build a Java or nodejs application using maven or npm, you may download many libraries from their respective external repositories. If at that time the repositories or access path have some performance issue, then the build process can fail or be delayed more than usual. This means that if your build depends on an external service or resource, it’s easy to suffer negative effects from that regardless of your local system’s status. So it may be best to consider creating a local repository (maven, npm, git and so on) to ensure reliable and stable performance for your builds. Or you can reuse previously downloaded dependencies and previously built artifacts by using Source-to-Image (S2I) incremental builds, if your image registry performance is enough to pull previously-built images for every incremental build.

* Performing Source-to-Image (S2I) incremental builds

```yaml
strategy:
  sourceStrategy:
    from:
      kind: "ImageStreamTag"
      name: "incremental-image:latest" 
    incremental: true
```

And building with Dockerfile can optimize layered caches to decrease image layer pull and push time. In the following example, it decreased the update size because splitting into layers is based on change frequency. To rephrase it, any unchanged layers are cached.

![layer_cache](https://github.com/bysnupy/blog/blob/master/kubernetes/layer_cache.png)

* This image has small layers but the 35M layer is always updated when the image build is conducted.

```command
# podman history 31952aa275b83b9108eeffa763f7f1f531ec0830e79981755908ad6dfaa03e1b
ID             CREATED          CREATED BY                                      SIZE      COMMENT
31952aa275b8   52 seconds ago   /bin/sh -c #(nop) ENTRYPOINT ["tail","-f",...   0B        
<missing>      57 seconds ago   /bin/sh -c #(nop) COPY resources /resources     35.7MB    
<missing>      5 weeks ago                                                      20.48kB   
<missing>      5 weeks ago                                                      107.1MB   Imported from -

# podman push docker-registry.default.svc:5000/openshift/test:latest
Getting image source signatures
Copying blob 3698255bccdb done
Copying blob a066f3d73913 skipped: already exists
Copying blob 26b543be03e2 skipped: already exists
Copying config 31952aa275 done
Writing manifest to image destination
Copying config 31952aa275 done
Writing manifest to image destination
Storing signatures
```

* But this image has more layers than the one above, the 5M layer is the only one that is updated. Other unchanged layers are cached.
```command
# buildah bud --layers .
STEP 1: FROM registry.redhat.io/ubi8/ubi-minimal:latest
STEP 2: COPY resources/base /resources/base
--> Using cache da63ee05ff89cbec02e8a6ac89c287f550337121b8b401752e98c53b22e4fea7
STEP 3: COPY resources/extra /resources/extra
--> Using cache 9abc7eee3e705e4999a7f2ffed09d388798d21d1584a5f025153174f1fa161b3
STEP 4: COPY resources/user /resources/user
b9cef39450b5e373bd4da14f446b6522e0b46f2aabac2756ae9ce726d240e011
STEP 5: ENTRYPOINT ["tail","-f","/dev/null"]
STEP 6: COMMIT
72cc8f59b6cd546d8fb5c3c5d82321b6d14bf66b91367bc5ca403eb33cfcdb15

# podman tag 72cc8f59b6cd546d8fb5c3c5d82321b6d14bf66b91367bc5ca403eb33cfcdb15 docker-registry.default.svc:5000/openshift/test:latest

# podman history 72cc8f59b6cd546d8fb5c3c5d82321b6d14bf66b91367bc5ca403eb33cfcdb15
ID             CREATED              CREATED BY                                      SIZE      COMMENT
72cc8f59b6cd   About a minute ago   /bin/sh -c #(nop) ENTRYPOINT ["tail","-f",...   0B        
<missing>      About a minute ago   /bin/sh -c #(nop) COPY resources/user /res...   5.245MB   
<missing>      2 minutes ago        /bin/sh -c #(nop) COPY resources/extra /re...   20.01MB   
<missing>      2 minutes ago        /bin/sh -c #(nop) COPY resources/base /res...   10.49MB   
<missing>      5 weeks ago                                                          20.48kB   
<missing>      5 weeks ago                                                          107.1MB   Imported from -

# podman push docker-registry.default.svc:5000/openshift/test:latest
Getting image source signatures
Copying blob aa6eb6fda701 done
Copying blob 26b543be03e2 skipped: already exists
Copying blob a066f3d73913 skipped: already exists
Copying blob 822ae69b69df skipped: already exists
Copying blob 7c5c2aefa536 skipped: already exists
Copying config 72cc8f59b6 done
Writing manifest to image destination
Copying config 72cc8f59b6 done
Writing manifest to image destination
Storing signatures
```

You should check build logs when troubleshooting the build process, because Kubernetes can only detect whether a build pod completed successfully or not.

## Application Pod

Like the Build Pod, all control passes to the application pod, once application pod is created, so most of the work is through the application implementation. If the application depends on external services and resources to initialize, such as DB connection pooling, KVS, and other API connections.

For instance, if DB server that uses connection pooling has performance issues or reaches the maximum connection count while the application pod is starting, the application pod initialization can be delayed more than expected. So if your application has external dependencies, you should also check to see whether they are running well. And if Readiness Probes and Liveness Probes are configured for your application pod, you should set initialDelaySeconds and periodSeconds large enough for your application pod to initialize. If it’s too short initialDelaySeconds and periodSeconds to check the application states, your application will be restarted repeatedly and may result in a delay or failure to deploy the application pod.

* Monitoring container health

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

Lastly, I recommend to keep concurrent build and deployment tasks in one cycle as appropriate numbers for suppressing resource issues. As you can see here, these tasks are chained with other tasks and it will iterate automatically through their life cycles, so it would happen more workload than you expected. Usually the compile (build) and application initialization (deployment) processes are CPU intensive tasks. So you may need to evaluate how many concurrent build and deployment tasks are possible to ensure you have a stable cluster before scheduling new tasks.

## Conclusion

We took a look at how each part affect each other within build and deployment processes. I focused on the configuration and other information related only to build and deployment performance topics in the hopes that this information will help your understanding about the interaction between the build and deployment phases of applications on OpenShift. I hope it’s helpful for your stable system management. Thank you for reading.
