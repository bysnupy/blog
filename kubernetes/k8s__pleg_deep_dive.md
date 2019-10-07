# "PLEG is not healthy" deep dive

## Introduction

The PLEG(Pod Lifecycle Event Generator) in kubelet(Kubernetes) convert the container runtime states with each matched pod-level event and maintain the pod cache up-to-date. This PLEG keep watching health state as calling `Healty()` function periodically by `runtimeState`. If the relistTime is greater than 3 minutes, "PLEG is not healthy" is shown.

* `Healthy()` is implementation for PLEG health check method.
```go
// pkg/kubelet/pleg/generic.go - Healthy()

// The threshold needs to be greater than the relisting period + the
// relisting time, which can vary significantly. Set a conservative
// threshold to avoid flipping between healthy and unhealthy.
relistThreshold = 3 * time.Minute
...
func (g *GenericPLEG) Healthy() (bool, error) {
	relistTime := g.getRelistTime()
	elapsed := g.clock.Since(relistTime)
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistThreshold)
	}
	return true, nil
}
```

* The `Healthy()` is registered `runtimeState` by `addHealthCheck` as "PLEG" name.
```go
// pkg/kubelet/kubelet.go - NewMainKubelet()
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
...
		klet.runtimeState.addHealthCheck("PLEG", klet.pleg.Healthy)
```

* The registered `healthCheck`(`Healthy()`) is evaluated periodically by `runtimeState.runtimeErrors()` in `syncLoop()`.
```go
// pkg/kubelet/kubelet.go - syncLoop()
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
...
		for {
				if rs := kl.runtimeState.runtimeErrors(); len(rs) != 0 {
					glog.Infof("skipping pod synchronization - %v", rs)
					// exponential backoff
					time.Sleep(duration)
					duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
					continue
				}

// pkg/kubelet/runtime.go - runtimeErrors()
func (s *runtimeState) runtimeErrors() []string {
...
		for _, hc := range s.healthChecks {
			if ok, err := hc.fn(); !ok {
				ret = append(ret, fmt.Sprintf("%s is not healthy: %v", hc.name, err))
			}
		}
```

If `Healthy()` is failed, we can see logs like "PLEG is not healthy: pleg was last seen active 3m1.1111s ago; threshold is 3m0s".


## Review `relist()`
I'd like to follow the `relist` to examine why "PLEG is unheathy" happen in the process from now.

* `relist()` is called periodically by goroutine, even though we set the period to 1s(`plegRelistPeriod`), the `relist()` itself can take more than 1s to finish if the container runtime responds slowly and/or when there are many container changes in one cycle. Next `relist()` can call after previous one is complete.

```go
// pkg/kubelet/kubelet.go - NewMainKubelet()

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
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
...
		klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})

// pkg/kubelet/pleg/generic.go - Start()

// Start spawns a goroutine to relist periodically.
func (g *GenericPLEG) Start() {
	go wait.Until(g.relist, g.relistPeriod, wait.NeverStop)
}
```

We start to review how `relist()` process is conducted from now.

```go
// relist queries the container runtime for list of pods/containers, compare
// with the internal pods/containers, and generates events accordingly.
func (g *GenericPLEG) relist() {
	glog.V(5).Infof("GenericPLEG: Relisting")
```

* `lastRelistTime` is used as `kubelet_pleg_relist_interval_microseconds` Prometheus metrics for interval in microseconds between relisting in PLEG.
  And `relist()` completion elapsed time is used as `kubelet_pleg_relist_latency_microseconds` Prometheus metrics for Latency in microseconds for relisting pods in PLEG.
```go
	if lastRelistTime := g.getRelistTime(); !lastRelistTime.IsZero() {
		metrics.PLEGRelistInterval.Observe(metrics.SinceInMicroseconds(lastRelistTime))
	}

	timestamp := g.clock.Now()
	defer func() {
		metrics.PLEGRelistLatency.Observe(metrics.SinceInMicroseconds(timestamp))
	}()
```

* Get all pods(included stopped pods by `GetPods(true)`) from container runtime.
```go
	// Get all the pods.
	podList, err := g.runtime.GetPods(true)
	if err != nil {
		glog.Errorf("GenericPLEG: Unable to retrieve pods: %v", err)
		return
	}
```

* For example, `GetPods()` is get the all pods using remote calls.
```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go - GetPods()

// GetPods returns a list of containers grouped by pods. The boolean parameter
// specifies whether the runtime returns all containers including those already
// exited and dead containers (used for garbage collection).
func (m *kubeGenericRuntimeManager) GetPods(all bool) ([]*kubecontainer.Pod, error) {
	pods := make(map[kubetypes.UID]*kubecontainer.Pod)
	sandboxes, err := m.getKubeletSandboxes(all)

// pkg/kubelet/kuberuntime/kuberuntime_sandbox.go - getKubeletSandboxes()

// getKubeletSandboxes lists all (or just the running) sandboxes managed by kubelet.
func (m *kubeGenericRuntimeManager) getKubeletSandboxes(all bool) ([]*runtimeapi.PodSandbox, error) {
	var filter *runtimeapi.PodSandboxFilter
	if !all {
		readyState := runtimeapi.PodSandboxState_SANDBOX_READY
		filter = &runtimeapi.PodSandboxFilter{
			State: &runtimeapi.PodSandboxStateValue{
				State: readyState,
			},
		}
	}

	resp, err := m.runtimeService.ListPodSandbox(filter)
	if err != nil {
		glog.Errorf("ListPodSandbox failed: %v", err)
		return nil, err
	}

	return resp, nil
}

// pkg/kubelet/remote/remote_runtime.go - ListPodSandbox()

// ListPodSandbox returns a list of PodSandboxes.
func (r *RemoteRuntimeService) ListPodSandbox(filter *runtimeapi.PodSandboxFilter) ([]*runtimeapi.PodSandbox, error) {
	ctx, cancel := getContextWithTimeout(r.timeout)
	defer cancel()

	resp, err := r.runtimeClient.ListPodSandbox(ctx, &runtimeapi.ListPodSandboxRequest{
		Filter: filter,
	})
	if err != nil {
		glog.Errorf("ListPodSandbox with filter %+v from runtime service failed: %v", filter, err)
		return nil, err
	}

	return resp.Items, nil
}
```


* At this point, time of last relist time updates as current timestamp.
  In other words, the elasped time in `Healthy()` can be evaluated as starting point at this updated timestamp, if `GetPods()` processed without any latency.
```go
	g.updateRelistTime(timestamp)
```

* All pods gotten from container runtime are setting as current pods, and to compare old and current pods to generate/update related each matched event.
```go
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

* If there are pods to be updated events and states, the related pod cache should be updated by `updateCache()`.
```go
	var needsReinspection map[types.UID]*kubecontainer.Pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}

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

				// make sure we try to reinspect the pod during the next relisting
				needsReinspection[pid] = pod

				continue
			} else if _, found := g.podsToReinspect[pid]; found {
				// this pod was in the list to reinspect and we did so because it had[0] `GetPods()` is get the all pods using remote calls.
```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go - GetPods()

// GetPods returns a list of containers grouped by pods. The boolean parameter
// specifies whether the runtime returns all containers including those already
// exited and dead containers (used for garbage collection).
func (m *kubeGenericRuntimeManager) GetPods(all bool) ([]*kubecontainer.Pod, error) {
	pods := make(map[kubetypes.UID]*kubecontainer.Pod)
	sandboxes, err := m.getKubeletSandboxes(all)

// pkg/kubelet/kuberuntime/kuberuntime_sandbox.go - getKubeletSandboxes()

// getKubeletSandboxes lists all (or just the running) sandboxes managed by kubelet.
func (m *kubeGenericRuntimeManager) getKubeletSandboxes(all bool) ([]*runtimeapi.PodSandbox, error) {
	var filter *runtimeapi.PodSandboxFilter
	if !all {
		readyState := runtimeapi.PodSandboxState_SANDBOX_READY
		filter = &runtimeapi.PodSandboxFilter{
			State: &runtimeapi.PodSandboxStateValue{
				State: readyState,
			},
		}
	}

	resp, err := m.runtimeService.ListPodSandbox(filter)
	if err != nil {
		glog.Errorf("ListPodSandbox failed: %v", err)
		return nil, err
	}

	return resp, nil
}

// pkg/kubelet/remote/remote_runtime.go - ListPodSandbox()

// ListPodSandbox returns a list of PodSandboxes.
func (r *RemoteRuntimeService) ListPodSandbox(filter *runtimeapi.PodSandboxFilter) ([]*runtimeapi.PodSandbox, error) {
	ctx, cancel := getContextWithTimeout(r.timeout)
	defer cancel()

	resp, err := r.runtimeClient.ListPodSandbox(ctx, &runtimeapi.ListPodSandboxRequest{
		Filter: filter,
	})
	if err != nil {
		glog.Errorf("ListPodSandbox with filter %+v from runtime service failed: %v", filter, err)
		return nil, err
	}

	return resp.Items, nil
}
``` events, so remove it
				// from the list (we don't want the reinspection code below to inspect it a second time in
				// this relist execution)
				delete(g.podsToReinspect, pid)
			}
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

* Any pods that failed in the previous loop are try to update cache again.
```go
	if g.cacheEnabled() {
		// reinspect any pods that failed inspection during the previous relist
		if len(g.podsToReinspect) > 0 {
			glog.V(5).Infof("GenericPLEG: Reinspecting pods that previously failed inspection")
			for pid, pod := range g.podsToReinspect {
				if err := g.updateCache(pod, pid); err != nil {
					glog.Errorf("PLEG: pod %s/%s failed reinspection: %v", pod.Name, pod.Namespace, err)
					needsReinspection[pid] = pod
				}
			}
		}

		// Update the cache timestamp.  This needs to happen *after*
		// all pods have been properly updated in the cache.
		g.cache.UpdateTime(timestamp)
	}

	// make sure we retain the list of pods that need reinspecting the next time relist is called
	g.podsToReinspect = needsReinspection
}
```

* `updateCache()` called each pod which is changed states with multiple remote calls to update through `GetPodStatus()`.
```go
// pkg/kubelet/pleg/generic.go - updateCache()

func (g *GenericPLEG) updateCache(pod *kubecontainer.Pod, pid types.UID) error {
	if pod == nil {
		// The pod is missing in the current relist. This means that
		// the pod has no visible (active or inactive) containers.
		glog.V(4).Infof("PLEG: Delete status for pod %q", string(pid))
		g.cache.Delete(pid)
		return nil
	}
	timestamp := g.clock.Now()
	// TODO: Consider adding a new runtime method
	// GetPodStatus(pod *kubecontainer.Pod) so that Docker can avoid listing
	// all containers again.
	status, err := g.runtime.GetPodStatus(pod.ID, pod.Name, pod.Namespace)
	glog.V(4).Infof("PLEG: Write status for %s/%s: %#v (err: %v)", pod.Name, pod.Namespace, status, err)
	if err == nil {
		// Preserve the pod IP across cache updates if the new IP is empty.
		// When a pod is torn down, kubelet may race with PLEG and retrieve
		// a pod status after network teardown, but the kubernetes API expects
		// the completed pod's IP to be available after the pod is dead.
		status.IP = g.getPodIP(pid, status)
	}

	g.cache.Set(pod.ID, status, err, timestamp)
	return err
}
```

* The `GetPodStatus()` get the information using `PodSandboxStatus()` and `getPodContainerStatuses()` which are calling remote calls.
```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go - GetPodStatus()

// GetPodStatus retrieves the status of the pod, including the
// information of all containers in the pod that are visible in Runtime.
func (m *kubeGenericRuntimeManager) GetPodStatus(uid kubetypes.UID, name, namespace string) (*kubecontainer.PodStatus, error) {
	// Now we retain restart count of container as a container label. Each time a container
	// restarts, pod will read the restart count from the registered dead container, increment
	// it to get the new restart count, and then add a label with the new restart count on
	// the newly started container.
	// However, there are some limitations of this method:
	//	1. When all dead containers were garbage collected, the container status could
	//	not get the historical value and would be *inaccurate*. Fortunately, the chance
	//	is really slim.
	//	2. When working with old version containers which have no restart count label,
	//	we can only assume their restart count is 0.
	// Anyhow, we only promised "best-effort" restart count reporting, we can just ignore
	// these limitations now.
	// TODO: move this comment to SyncPod.
	podSandboxIDs, err := m.getSandboxIDByPodUID(uid, nil)
  ...
	for idx, podSandboxID := range podSandboxIDs {
		podSandboxStatus, err := m.runtimeService.PodSandboxStatus(podSandboxID)
    ...
	}

	// Get statuses of all containers visible in the pod.
	containerStatuses, err := m.getPodContainerStatuses(uid, name, namespace)
...
}
```

* For example, we can see `GetPodNetworkStatus()` which be included `PodSandboxStatus()` tracestack is getting lock during process either.
```go
// pkg/kubelet/dockershim/network/plugins.go

func (pm *PluginManager) GetPodNetworkStatus(podNamespace, podName string, id kubecontainer.ContainerID) (*PodNetworkStatus, error) {
  defer recordOperation("get_pod_network_status", time.Now())
  fullPodName := kubecontainer.BuildPodFullName(podName, podNamespace)
  pm.podLock(fullPodName).Lock()
  defer pm.podUnlock(fullPodName)

  netStatus, err := pm.plugin.GetPodNetworkStatus(podNamespace, podName, id)
  if err != nil {
    return nil, fmt.Errorf("NetworkPlugin %s failed on the status hook for pod %q: %v", pm.plugin.Name(), fullPodName, err)
  }

  return netStatus, nil
}
```

* For example, `getPodContainerStatuses()` get container list and status using remote calls.
```go
// getPodContainerStatuses gets all containers' statuses for the pod.
func (m *kubeGenericRuntimeManager) getPodContainerStatuses(uid kubetypes.UID, name, namespace string) ([]*kubecontainer.ContainerStatus, error) {
  // Select all containers of the given pod.
  containers, err := m.runtimeService.ListContainers(&runtimeapi.ContainerFilter{
    LabelSelector: map[string]string{types.KubernetesPodUIDLabel: string(uid)},
  })
  ...

  statuses := make([]*kubecontainer.ContainerStatus, len(containers))
  // TODO: optimization: set maximum number of containers per container name to examine.
  for i, c := range containers {
    status, err := m.runtimeService.ContainerStatus(c.Id)
    ...
  }
}
```

## Conclusion

PLEG get many information through container runtime, and kubelet runtime request timeout is 2min by default, so I think if there are some unresponsive pods, "PLEG is not healthy" error is triggered. And the update cache is processing through single loop, it takes time the more containers. Additionally the more containers are easy to generate more events, it also take time more to update the caches.


[A] Kubelet: Pod Lifecycle Event Generator (PLEG)
    [https://github.com/bysnupy/community/blob/master/contributors/design-proposals/node/pod-lifecycle-event-generator.md]

[B] Kubelet: Runtime Pod Cache
    [https://github.com/bysnupy/community/blob/master/contributors/design-proposals/node/runtime-pod-cache.md]
