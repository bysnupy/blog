# How to keep build and deployment stable performance on OpenShift ?

In this article, I will introduce some helpful common tips to manage reliably build and deployment on OpenShift. If you have been experienced sudden performance degradation of build and deployment on OpenShift, it may be helpful to troubleshoot your cluster. We start to review whole processes from build to deployment, and then cover each process for more details. Basically OpenShift 4.2(Kubernetes 1.14) is based on here.
Processes from build to deployment

When CI/CD or user triggers the build for a Pod deployment, the processes will proceed like the following image that is simplified for readability.

![build_deploy_process](https://github.com/bysnupy/blog/blob/master/kubernetes/build_deploy_performance.png)

As you can see, all parts are not controlled over by only Kubernetes, usually main processes for build and deployment depend on own configuration and external dependencies in the systems. We can split into each process by what component have responsibility to manage as follows.

Responsibility|Component
-|-
Schedule and Control a Pod | Kubernetes
Manage images | Image Registry
Build image for an application | build Pod
Deploy an application|application Pod

## Kubernetes

In this process, Kubernetes provides resources based on `buildConfig` and `deploymentConfig`. Usually build and deploy will be failed before starting their work, if any trouble happen in handling resources, such as volume mount failure, scheduling pending and so on. And you can also find the reasons easily in the event logs.
So if build and deployment is successful, even though the time took more than before, it would not be Kubernetes scheduling and control problem.
For clarifying the performance issue, we should ensure `buildConfig` and `deploymentConfig` are configured with enough resource requests through the following examples.

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

Additionally, you should add `kube-reserved` and `system-reserved` to provide more reliable scheduling and minimize node resource overcommitment in all nodes.
Refer Allocating resources for nodes in an OpenShift Container Platform cluster for more details.

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

If you have more than 1000 nodes, you can consider to tune percentageOfNodesToScore for scheduler performance tuning, this feature state is beta as of Kubernetes 1.14(OpenShift 4.2).

##Image Registry

This component work on image push and pull tasks by build and deployment.
First of all, you should consider to adopt appropriate storage and network enough to
process required traffic and IO for maximum concurrent image pull and push you assumed.
Usually object storage is recommended, because it is atomic, meaning the data is either written completely or not written at all, even if there is a failure during the write.
And it can share a volume with other duplicated registry as RWX(ReadWriteMany) mode.
Further information is here: Recommended configurable storage technology.
You can also consider `ImagePullPolicy` to reduce pull/push workload as follows.

* Image pull policy

Value|Description
-|-
Always|Always pull the image.
IfNotPresent|Only pull the image if it does not already exist on the node.
Never|Never pull the image

## Build Pod

When build Pod is created and start to build application, all control passes to the build Pod, and work through build configurations are defined by a `BuildConfig`.
Depending on how you choose to create your application and how to provide source content to build and operate on affect build performance.
For instance, if you build Java or nodejs application using `maven` or `npm`, you may download many libraries from their external repositories.
At that time the repositories or access path have some performance issue, then the build process will be failed or delayed more than usual time.
It means if your build depend on external service or resource, it's easy to affect from that regardless of your system status. So you can consider to create local repository (maven, npm, git and so on) for reliable and stable performance for build. Or you can reuse previously downloaded dependencies, previously built artifacts as using Source-to-Image (S2I) incremental builds, if your image registry performance is enough to pull previously-built images for every incremental builds.

* Performing Source-to-Image (S2I) incremental builds

```yaml
strategy:
  sourceStrategy:
    from:
      kind: "ImageStreamTag"
      name: "incremental-image:latest" 
    incremental: true
```

And build using `Dockerfile` can optimize layered caches to decrease pull/push time.
Let's take a look the following example, it decreased the update size as splitting into layers which is based on change frequency. To rephrase it, other unchanged layers are cached.

![layer_cache](https://github.com/bysnupy/blog/blob/master/kubernetes/layer_cache.png)

* This image has small layers but, the 35M layer is always updated when the image build is conducted.

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

* But this image has more layers then above one, the 5M layer is only updated. Other unchanged layers are cached.
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

You should check build logs for troubleshooting in a build process, because Kubernetes can only detect build Pod is complete successfully or not.

## Application Pod

Like the Build Pod, all controls passes to the application Pod, when application Pod is created, and work through application implementation. If the application depends on external services and resources to initialize, such as DB connection pooling, KVS and other API connections.
For instance, if DB server that is required to connection pooling have performance issues or reach the maximum connection count while application Pod is starting, 
the application Pod initialization can delay more than you expected, so if your application have external dependencies, you also check them whether they are running well. And if `Readiness Probes` and `Liveness Probes` is configured for your application Pod, you should set `initialDelaySeconds` and `periodSeconds` enough to your application Pod initialized. If it's too short `initialDeplaySeonds` and `periodSeconds` to check the application states, your application will be restarted uselessly, it result in delay to deploy application Pod.

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

Lastly, I recommend to keep concurrent build and deployment tasks in one cycle as appropriate numbers for suppressing resource issues. 
As you seen here, these tasks are chained with other tasks and it will iterate automatically through their lifecycles, so it would happen more workload than you expected. Usually compile(build) and application initialization(deployment) process is a CPU aggressive task, you may need to evaluate what count concurrent build and deployment tasks is available and stable on your cluster before scheduling their tasks.

## Conclusion

We took a look how each part affect each other within build and deployment processes.
You may already know each configuration I introduced above and it's nothing special.
As I introduced the configuration and information set for only build and deployment performance topic for help your understanding about interaction on the build and deployment here. I hope it's helpful for your stable system management. Thank you for reading.
