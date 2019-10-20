# Title: Pod Lifecycle Event Generator (PLEG) on OpenShift

Red HatのDaein(デイン)です。ノードホストのJournalログから次のようなメッセージを一度見たことがあると思います。
```command
kubelet.go:1865] SyncLoop (PLEG): "CONTAINER_NAME(...)", event: &pleg.PodLifecycleEvent{ID:"...", Type:"ContainerDied", Data:"..."}
```
