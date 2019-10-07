# Title: "PLEG is not healthy" deep dive

# What is PLEG ?

In this article, I'd like to talk about "PLEG is not healthy" issue, which is based on the implementation.
Let me explain Pod Lifecycle Event Generator(PLEG) simply here, this module in kubelet(Kubernetes) convert the container runtime states with each matched pod-level event and maintain the pod cache up-to-date.

# "PLEG is not healthy": How this issue is triggered ? 
In order to  answer this question, I'd like to walk through a few functions for PLEG health check tasks from now.
Kubelet keep checking PLEG health by calling `Healthy()` periodically in `SyncLoop()` as follows. 

* Healthy(): check whether relist process(the PLEG key task) completes within 3 minutes.
```go
// pkg/kubelet/pleg/generic.go - Healthy()
// the source codes in this article has been modified simply for readability.

relistThreshold = 3 * time.Minute

func (g *GenericPLEG) Healthy() (bool, error) {
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistThreshold)
	}
}
```

* SyncLoop(): check the PLEG health periodically using `runtimeErrors()` as follows. 


![PLEG_process_flow](https://octodex.github.com/images/yaktocat.png)
