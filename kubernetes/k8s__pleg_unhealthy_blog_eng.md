# Title: Understanding "PLEG is not healthy"

In this post, I'd like to talk about "PLEG is not healthy" issue, sometimes which is leading "NodeNotReady" status. 
As understanding how the PLEG works, it will be helpful for the troubleshooting around this issue.

## What is the PLEG ?

PLEG is stand for Pod Lifecycle Event Generator, 
this module in kubelet(Kubernetes) convert accordingly the container runtime states with each matched pod-level event,
and maintain the pod cache up-to-date by applying changes.

We will take a look around the red dot line part from below process image.
![original_pleg_flow_image](https://github.com/bysnupy/blog/blob/master/kubernetes/orig-pleg.png)

The original image is here: [Kubelet: Pod Lifecycle Event Generator (PLEG)](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/pod-lifecycle-event-generator.md).

## How does "PLEG is not healthy" happen ?
Kubelet keeps checking PLEG health by calling `Healthy()` periodically in `SyncLoop()` as follows. 

`Healthy()` checks whether "relist" process(the PLEG key task) completes within 3 minutes.
This function is added to `runtimeState` as "PLEG", and it's calling periodically from `SyncLoop`(called every 10s by default). 
If the "relist" process take times more than 3 minutes, "PLEG is not healthy" issue happens through this stack processes.
We got to know the "relist" is key task for this issue at now. 
I'll walk you through related source codes based on Kubernetes 1.11(OpenShift 3.11) in each part to help your more understanding.
Don't worry if you are not familiar with Go syntax, it's enough to read comments around codes. 
In addition to I also explain summary before the codes and snipped less important things from the source codes for readability either.

```go
//// pkg/kubelet/pleg/generic.go - Healthy()

// The threshold needs to be greater than the relisting period + the
// relisting time, which can vary significantly. Set a conservative
// threshold to avoid flipping between healthy and unhealthy.
relistThreshold = 3 * time.Minute
:
func (g *GenericPLEG) Healthy() (bool, error) {
	relistTime := g.getRelistTime()
	elapsed := g.clock.Since(relistTime)
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistThreshold)
	}
	return true, nil
}

//// pkg/kubelet/kubelet.go - NewMainKubelet()
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration, ...
:
  klet.runtimeState.addHealthCheck("PLEG", klet.pleg.Healthy)

//// pkg/kubelet/kubelet.go - syncLoop()
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
:
// The resyncTicker wakes up kubelet to checks if there are any pod workers
// that need to be sync'd. A one-second period is sufficient because the
// sync interval is defaulted to 10s.
:
  const (
    base   = 100 * time.Millisecond
    max    = 5 * time.Second
    factor = 2
  )
  duration := base
	for {
		if rs := kl.runtimeState.runtimeErrors(); len(rs) != 0 {
			glog.Infof("skipping pod synchronization - %v", rs)
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
    :
  }
:
}

//// pkg/kubelet/runtime.go - runtimeErrors()
func (s *runtimeState) runtimeErrors() []string {
:
	for _, hc := range s.healthChecks {
		if ok, err := hc.fn(); !ok {
			ret = append(ret, fmt.Sprintf("%s is not healthy: %v", hc.name, err))
		}
	}
:
}
```

## Review "relist"

Take a look more details about "relist" function as follows. 
Specifically watch out carefully the remote process calls with other parts and check how to process the getting data,
because those parts are easy to bottleneck usually.

![PLEG_process_flow](https://github.com/bysnupy/blog/blob/master/kubernetes/PLEG.png)

In the same order above flow chart, check the process and implementations in the "relist". Refer [here](https://github.com/openshift/origin/blob/release-3.11/vendor/k8s.io/kubernetes/pkg/kubelet/pleg/generic.go#L180-L284) for full source codes.

Even though "relist" is setting as calling every 1s, however itself can take more than 1s to finish, 
if the container runtime responds slowly and/or when there are many container changes in one cycle.
So next "relist" can call after previous one is complete. 
For example, if "relist" takes time 5s to complete, then next relist time is after 6s(1s + 5s).

```go
//// pkg/kubelet/kubelet.go - NewMainKubelet()

// Generic PLEG relies on relisting for discovering container events.
// A longer period means that kubelet will take longer to detect container
// changes and to update pod status. On the other hand, a shorter period
// will cause more frequent relisting (e.g., container runtime operations),
// leading to higher cpu usage.
// Note that even though we set the period to 1s, the relisting itself can
// take more than 1s to finish if the container runtime responds slowly
// and/or when there are many container changes in one cycle.
plegRelistPeriod = time.Second * 1

// NewMainKubelet instantiates a new Kubelet object along with all the required internal modules.
// No initialization of Kubelet and its modules should happen here.
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration, ... 
:
	klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})

//// pkg/kubelet/pleg/generic.go - Start()

// Start spawns a goroutine to relist periodically.
func (g *GenericPLEG) Start() {
	go wait.Until(g.relist, g.relistPeriod, wait.NeverStop)
}

//// pkg/kubelet/pleg/generic.go - relist()
func (g *GenericPLEG) relist() {
... We focus here ...
}
```

The function process starts as recording some metrics for Prometheus, such like `kubelet_pleg_relist_latency_microseconds`.
And then take all pods(included stopped pods) list from container runtime using CRI interface for getting current pods status.
This pods list is using for comparison with previous pods list to check changes and the matched pod-level events are generated along the changed states.

```go
//// pkg/kubelet/pleg/generic.go - relist()
  :
  // get a current timestamp
  timestamp := g.clock.Now()

  // kubelet_pleg_relist_latency_microseconds for prometheus metrics
	defer func() {
		metrics.PLEGRelistLatency.Observe(metrics.SinceInMicroseconds(timestamp))
	}()
  
  // Get all the pods.
	podList, err := g.runtime.GetPods(true)
  :
```

* Trace `GetPods()` call stack details are here.
```go
//// pkg/kubelet/kuberuntime/kuberuntime_manager.go - GetPods()

// GetPods returns a list of containers grouped by pods. The boolean parameter
// specifies whether the runtime returns all containers including those already
// exited and dead containers (used for garbage collection).
func (m *kubeGenericRuntimeManager) GetPods(all bool) ([]*kubecontainer.Pod, error) {
	pods := make(map[kubetypes.UID]*kubecontainer.Pod)
	sandboxes, err := m.getKubeletSandboxes(all)
:
}

//// pkg/kubelet/kuberuntime/kuberuntime_sandbox.go - getKubeletSandboxes()

// getKubeletSandboxes lists all (or just the running) sandboxes managed by kubelet.
func (m *kubeGenericRuntimeManager) getKubeletSandboxes(all bool) ([]*runtimeapi.PodSandbox, error) {
:
	resp, err := m.runtimeService.ListPodSandbox(filter)
:
}

//// pkg/kubelet/remote/remote_runtime.go - ListPodSandbox()

// ListPodSandbox returns a list of PodSandboxes.
func (r *RemoteRuntimeService) ListPodSandbox(filter *runtimeapi.PodSandboxFilter) ([]*runtimeapi.PodSandbox, error) {
:
	resp, err := r.runtimeClient.ListPodSandbox(ctx, &runtimeapi.ListPodSandboxRequest{
:
	return resp.Items, nil
}
```

After getting all pods, last "relist" time is updated as current timestamp. In other words, `Healthy()` can be evaluated by this updated timestamp.

```go
//// pkg/kubelet/pleg/generic.go - relist()
  // update as a current timestamp
  g.updateRelistTime(timestamp)
```

As mentioned previously, after comparing between current and previous pods list, 
every matched pod-level event is generated by the differences/changes between both lists here.
`generateEvents()` generates matched pod-level events, such as "ContainerStarted", "ContainerDied" and so on, and then the events are updated by `updateEvents()`

```go
//// pkg/kubelet/pleg/generic.go - relist()

  pods := kubecontainer.Pods(podList)
  g.podRecords.setCurrent(pods)

  // Compare the old and the current pods, and generate events.
  eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
  for pid := range g.podRecords {
    oldPod := g.podRecords.getOld(pid)
    pod := g.podRecords.getCurrent(pid)
    // Get all containers in the old and the new pod.
    allContainers := getContainersFromPods(oldPod, pod)
    for _, container := range allContainers {
      events := computeEvents(oldPod, pod, &container.ID)
      for _, e := range events {
        updateEvents(eventsByPodID, e)
      }
    }
  }
```
* Trace `computeEvents()` call stack details are here.
```go
//// pkg/kubelet/pleg/generic.go - computeEvents()

func computeEvents(oldPod, newPod *kubecontainer.Pod, cid *kubecontainer.ContainerID) []*PodLifecycleEvent {
:
	return generateEvents(pid, cid.ID, oldState, newState)
}

//// pkg/kubelet/pleg/generic.go - generateEvents()

func generateEvents(podID types.UID, cid string, oldState, newState plegContainerState) []*PodLifecycleEvent {
:
	glog.V(4).Infof("GenericPLEG: %v/%v: %v -> %v", podID, cid, oldState, newState)
	switch newState {
	case plegContainerRunning:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerStarted, Data: cid}}
	case plegContainerExited:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}}
	case plegContainerUnknown:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerChanged, Data: cid}}
	case plegContainerNonExistent:
		switch oldState {
		case plegContainerExited:
			// We already reported that the container died before.
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerRemoved, Data: cid}}
		default:
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}, {ID: podID, Type: ContainerRemoved, Data: cid}}
		}
	default:
		panic(fmt.Sprintf("unrecognized container state: %v", newState))
	}
}
```

The last process is If there are events associated with a pod, we should update the podCache as follows.
`updateCache()` will inspect each pod and update it one by one in single loop, 
so if many pods changed during the same "relist" period this process can be bottleneck.
Lastly updated new pod lifecycle event is sent to eventChannel after updates.

Tracing call stack details are not so important for understanding, 
just understand some remote clients are called per a pod for getting pod inspect information by conditions.
I think this way will be increasing latency for proportional to pod numbers, because many pods are generated many events usually.

```go
//// pkg/kubelet/pleg/generic.go - relist()

  // If there are events associated with a pod, we should update the
  // podCache.
  for pid, events := range eventsByPodID {
    pod := g.podRecords.getCurrent(pid)
    if g.cacheEnabled() {
      // updateCache() will inspect the pod and update the cache. If an
      // error occurs during the inspection, we want PLEG to retry again
      // in the next relist. To achieve this, we do not update the
      // associated podRecord of the pod, so that the change will be
      // detect again in the next relist.
      // TODO: If many pods changed during the same relist period,
      // inspecting the pod and getting the PodStatus to update the cache
      // serially may take a while. We should be aware of this and
      // parallelize if needed.
      if err := g.updateCache(pod, pid); err != nil {
        glog.Errorf("PLEG: Ignoring events for pod %s/%s: %v", pod.Name, pod.Namespace, err)
        :
      }
      :
    }
    // Update the internal storage and send out the events.
    g.podRecords.update(pid)
    for i := range events {
      // Filter out events that are not reliable and no other components use yet.
      if events[i].Type == ContainerChanged {
        continue
      }
      g.eventChannel <- events[i]
    }
  }
```

* Trace `updateCache()` call stack details are here. multiple remote request is called by `GetPodStatus()` for pod inspect.
```go
//// pkg/kubelet/pleg/generic.go - updateCache()

func (g *GenericPLEG) updateCache(pod *kubecontainer.Pod, pid types.UID) error {
:
	timestamp := g.clock.Now()
	// TODO: Consider adding a new runtime method
	// GetPodStatus(pod *kubecontainer.Pod) so that Docker can avoid listing
	// all containers again.
	status, err := g.runtime.GetPodStatus(pod.ID, pod.Name, pod.Namespace)
  :
	g.cache.Set(pod.ID, status, err, timestamp)
	return err
}

//// pkg/kubelet/kuberuntime/kuberuntime_manager.go - GetPodStatus()

// GetPodStatus retrieves the status of the pod, including the
// information of all containers in the pod that are visible in Runtime.
func (m *kubeGenericRuntimeManager) GetPodStatus(uid kubetypes.UID, name, namespace string) (*kubecontainer.PodStatus, error) {
  :
	for idx, podSandboxID := range podSandboxIDs {
		podSandboxStatus, err := m.runtimeService.PodSandboxStatus(podSandboxID)
    :
	}

	// Get statuses of all containers visible in the pod.
	containerStatuses, err := m.getPodContainerStatuses(uid, name, namespace)
  :
}

//// pkg/kubelet/remote/remote_runtime.go - PodSandboxStatus()

// PodSandboxStatus returns the status of the PodSandbox.
func (r *RemoteRuntimeService) PodSandboxStatus(podSandBoxID string) (*runtimeapi.PodSandboxStatus, error) {
	ctx, cancel := getContextWithTimeout(r.timeout)
	defer cancel()
  
	resp, err := r.runtimeClient.PodSandboxStatus(ctx, &runtimeapi.PodSandboxStatusRequest{
		PodSandboxId: podSandBoxID,
	})
  :
	return resp.Status, nil
}

//// pkg/kubelet/dockershim/docker_sandbox.go - PodSandBoxStatus()

// PodSandboxStatus returns the status of the PodSandbox.
func (ds *dockerService) PodSandboxStatus(ctx context.Context, req *runtimeapi.PodSandboxStatusRequest) (*runtimeapi.PodSandboxStatusResponse, error) {
	podSandboxID := req.PodSandboxId

	r, metadata, err := ds.getPodSandboxDetails(podSandboxID)
  :
	var IP string
	// TODO: Remove this when sandbox is available on windows
	// This is a workaround for windows, where sandbox is not in use, and pod IP is determined through containers belonging to the Pod.
	if IP = ds.determinePodIPBySandboxID(podSandboxID); IP == "" {
		IP = ds.getIP(podSandboxID, r)
	}

	labels, annotations := extractLabels(r.Config.Labels)
	status := &runtimeapi.PodSandboxStatus{
		Id:          r.ID,
		State:       state,
		CreatedAt:   ct,
		Metadata:    metadata,
		Labels:      labels,
		Annotations: annotations,
		Network: &runtimeapi.PodSandboxNetworkStatus{
			Ip: IP,
		},
		Linux: &runtimeapi.LinuxPodSandboxStatus{
			Namespaces: &runtimeapi.Namespace{
				Options: &runtimeapi.NamespaceOption{
					Network: networkNamespaceMode(r),
					Pid:     pidNamespaceMode(r),
					Ipc:     ipcNamespaceMode(r),
				},
			},
		},
	}
	return &runtimeapi.PodSandboxStatusResponse{Status: status}, nil
}

//// pkg/kubelet/dockershim/docker_sandbox.go - getPodSandboxDetails()

// Returns the inspect container response, the sandbox metadata, and network namespace mode
func (ds *dockerService) getPodSandboxDetails(podSandboxID string) (*dockertypes.ContainerJSON, *runtimeapi.PodSandboxMetadata, error) {
	resp, err := ds.client.InspectContainer(podSandboxID)
  :
	return resp, metadata, nil
}

//// pkg/kubelet/dockershim/libdocker/kube_docker_client.go - InspectContainer()
func (d *kubeDockerClient) InspectContainer(id string) (*dockertypes.ContainerJSON, error) {
	ctx, cancel := d.getTimeoutContext()
	defer cancel()
	containerJSON, err := d.client.ContainerInspect(ctx, id)
  :
	return &containerJSON, nil
}

// getIP returns the ip given the output of `docker inspect` on a pod sandbox,
// first interrogating any registered plugins, then simply trusting the ip
// in the sandbox itself. We look for an ipv4 address before ipv6.
func (ds *dockerService) getIP(podSandboxID string, sandbox *dockertypes.ContainerJSON) string {
	if sandbox.NetworkSettings == nil {
		return ""
	}
	if networkNamespaceMode(sandbox) == runtimeapi.NamespaceMode_NODE {
		// For sandboxes using host network, the shim is not responsible for
		// reporting the IP.
		return ""
	}

	// Don't bother getting IP if the pod is known and networking isn't ready
	ready, ok := ds.getNetworkReady(podSandboxID)
	if ok && !ready {
		return ""
	}

	ip, err := ds.getIPFromPlugin(sandbox)
	if err == nil {
		return ip
	}
  :
}

//// pkg/kubelet/dockershim/docker_sandbox.go - getIPFromPlugin()

// getIPFromPlugin interrogates the network plugin for an IP.
func (ds *dockerService) getIPFromPlugin(sandbox *dockertypes.ContainerJSON) (string, error) {
  :
	networkStatus, err := ds.network.GetPodNetworkStatus(metadata.Namespace, metadata.Name, cID)
  :
	return networkStatus.IP.String(), nil
}

//// pkg/kubelet/dockershim/network/plugins.go - GetPodNetworkStatus()

func (pm *PluginManager) GetPodNetworkStatus(podNamespace, podName string, id kubecontainer.ContainerID) (*PodNetworkStatus, error) {
  defer recordOperation("get_pod_network_status", time.Now())
  fullPodName := kubecontainer.BuildPodFullName(podName, podNamespace)
  pm.podLock(fullPodName).Lock()
  defer pm.podUnlock(fullPodName)

  netStatus, err := pm.plugin.GetPodNetworkStatus(podNamespace, podName, id)
  :

  return netStatus, nil
}
```

We have taken a look the "relist" process through related source code and call stack trace.
I hope you can learn more details about PLEG and how to take/update the required data in the process.

## Conclusions
In my experience and searching, "PLEG is not healthy" can be happened by various causes, 
and I think there are many potential causes we do not run into yet around this issue either.
So I'd like to introduce the following past and usual cause for your additional information.

- Container runtime latency or timeout (performance degradation, deadlock, bugs ...) during remote requests
- Too many running pods for host resources, or too many running pods on high spec hosts to complete the relist within 3 min.
  As you checked in this post, events and latency is proportional to the pod numbers regardless host resources.
- [Deadlock in PLEG relist](https://github.com/kubergitetes/kubernetes/issues/72482), it has fixed as of Kubernetes 1.14.
- CNI bugs when getting a pod network status.


### References
- [Kubelet: Pod Lifecycle Event Generator (PLEG)](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/pod-lifecycle-event-generator.md)
- [Kubelet: Runtime Pod Cache](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/runtime-pod-cache.md)
- [relist() in kubernetes/pkg/kubelet/pleg/generic.go](https://github.com/openshift/origin/blob/release-3.11/vendor/k8s.io/kubernetes/pkg/kubelet/pleg/generic.go#L180-L284)
- [Past bug about CNI - PLEG is not healthy error, node marked NotReady](https://bugzilla.redhat.com/show_bug.cgi?id=1486914#c16)
