# Title: Understanding "PLEG is not healthy"

In this article, I'd like to talk about "PLEG is not healthy" issue, sometimes which is leading "NodeNotReady" status. 
As understanding how the PLEG works, it will be much helpful for the troubleshooting.

# What is the PLEG ?

PLEG is stand for Pod Lifecycle Event Generator, 
this module in kubelet(Kubernetes) convert accordingly the container runtime states with each matched pod-level event,
and maintain the pod cache up-to-date by applying changes.

# How does "PLEG is not healthy" happen ?
Kubelet keeps checking PLEG health by calling `Healthy()` periodically in `SyncLoop()` as follows. 

`Healthy()` checks whether "relist" process(the PLEG key task) completes within 3 minutes.
This function is added to `runtimeState` as "PLEG", and it's calling periodically from `SyncLoop`. 
If the "relist" process take times more than 3 minutes, "PLEG is not healthy" issue happens through this stack processes.
We got to know the "relist" is key task for this issue at now.

```go
// pkg/kubelet/pleg/generic.go - Healthy()
// the source codes in this article has been modified simply for readability.
relistThreshold = 3 * time.Minute

func (g *GenericPLEG) Healthy() (bool, error) {
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistThreshold)
	}
}

// pkg/kubelet/kubelet.go - NewMainKubelet()
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration, ... {
	klet.runtimeState.addHealthCheck("PLEG", klet.pleg.Healthy)
}

// pkg/kubelet/kubelet.go - syncLoop()
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	for {
		if rs := kl.runtimeState.runtimeErrors(); len(rs) != 0 {
			glog.Infof("skipping pod synchronization - %v", rs)
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
	}
}

// pkg/kubelet/runtime.go - runtimeErrors()
func (s *runtimeState) runtimeErrors() []string {
	for _, hc := range s.healthChecks {
		if ok, err := hc.fn(); !ok {
			ret = append(ret, fmt.Sprintf("%s is not healthy: %v", hc.name, err))
		}
	}
}
```

# Review "relist"

Take a look more details about "relist" function as follows. 
Specifically look at the remote process calls with other parts and check how to process the getting data,
because those parts are easy to bottleneck usually.

![PLEG_process_flow](https://github.com/bysnupy/blog/blob/master/kubernetes/PLEG.png)

In the same order above flow chart, check the process and implementations.

Even though "relist" is setting as calling every 1s, however itself can take more than 1s to finish, 
if the container runtime responds slowly and/or when there are many container changes in one cycle.
So next "relist" can call after previous one is complete. 
For example, if "relist" takes time 5s to complete, then next relist time is after 6s(1s + 5s).

```go
// pkg/kubelet/kubelet.go - NewMainKubelet()
plegRelistPeriod = time.Second * 1

func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration, ... {
	klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})
}
// pkg/kubelet/pleg/generic.go - Start()

// Start spawns a goroutine to relist periodically.
func (g *GenericPLEG) Start() {
	go wait.Until(g.relist, g.relistPeriod, wait.NeverStop)
}
```
