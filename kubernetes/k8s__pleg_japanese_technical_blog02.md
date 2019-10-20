# Title: Pod Lifecycle Event Generator (PLEG) on OpenShift

Red HatのDaein（デイン）です。ノードホストのJournalログで次のようなメッセージを一度は確認したことがあると思います。
```command
kubelet.go:1865] SyncLoop (PLEG): "CONTAINER_NAME(...)", event: &pleg.PodLifecycleEvent{ID:"...", Type:"ContainerStarted", Data:"..."}
```
このメッセージの中に含まれている謎の"PLEG"について見ていきたいと思います。

"PLEG"は"Pod Lifecycle Event Generator"の略語で"kubelet"に含まれているモジュールとして"container runtime"から"Pod"の現状を取得して
"Pod Lifecycle Event"に合わせて変換することとその変更された状態に合わせて"Pod Cache"を更新する作業を実施いています。

https://github.com/bysnupy/community/blob/master/contributors/design-proposals/node/pod-lifecycle-event-generator.md

https://github.com/bysnupy/community/blob/master/contributors/design-proposals/node/runtime-pod-cache.md

## PLEGの作業内容

先ほど話した通り"PLEG"はイベント生成のため現状稼働中のPodの変更を検知するために周期的に全ての"Pod"を一覧"container runtime"からプールしてその前後を比較することで確認しています。
この作業は次のイメージの"Relist"という点線で囲まれた一式の作業に含まれて変更内容に合わせて
