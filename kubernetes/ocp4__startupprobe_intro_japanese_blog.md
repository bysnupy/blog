# コンテナの初期起動フェーズをstartupProbeで監視 on OpenShift

Red HatでOpenShiftのサポートエンジニアをしているDaein（デイン）です。
OpenShift 4.5(Kubernetes 1.18)からstartupProbeがBeta機能としてデフォルトで利用できるようになりましたのでどのような機能であるか確認していきます。

## Probesとは

今まではlivenessProbeとreadinessProbeを利用して安定的なサービスが提供できるようにコンテナの起動状態を定期的にチェックして
正しく起動できなくなったコンテナを再起動させたり、外部からのアクセスを遮断させたりする機能になります。

詳しい内容や詳細は次のリンク先をご参照頂ければと思います。
https://docs.openshift.com/container-platform/4.5/applications/application-health.html
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

今回はstartupProbeという起動フェーズを別途監視できる追加機能にフォーカスしてみてみたいと思います。

## startupProbeが追加された背景

起動まである程度時間が必要なコンテナの場合はlivenessProbeのinitialDelaySeconds（コンテナ起動後、最初Probeが実施されるまでの秒数を指定）で
初期化が成功していたか確認して失敗した場合はコンテナを再起動して改めてコンテナの初期化を実施できるように構成する必要がありました。
ただし、livenessProbeはコンテナの起動時だけではなく全ライフサイクルをカバーする必要があるため、
アプリケーションの起動時間がより長くなったり、大きく変動したりするとそれに偏った設定が難しくなってしまいがちですが、その時startupProbeを利用して他のProbesと切り分けて設定できます。

Upstreamの関連文書は次の通りになります。
https://github.com/kubernetes/enhancements/blob/c872c41603f9822f6947256610feb2aae12b2253/keps/sig-node/20190221-livenessprobe-holdoff.md#summary

## startupProbeの設定と処理フロー
startupProbeは既存livenessProbeと全く同じ方法で設定できるため、特別な設定などが追加されたわけではありませんが、
startupProbeが設定された場合は、コンテナ起動後、他のprobesより先に評価されて成功と判定されないコンテナが再起動されて次のProbesが開始しない処理フローの違いがあります。

readinessProbeと違ってstartupProbeとlivenessProbeはsuccessThresholdを"1"しか指定できないのでご参考ください。
https://github.com/openshift/origin/blob/47c0e715581cfb08dd54dd53e37c49b9182f0298/vendor/k8s.io/kubernetes/pkg/apis/core/validation/validation.go#L2739-L2747
```golang
  // Liveness-specific validation
  if ctr.LivenessProbe != nil && ctr.LivenessProbe.SuccessThreshold != 1 {
    allErrs = append(allErrs, field.Invalid(idxPath.Child("livenessProbe", "successThreshold"), ctr.LivenessProbe.SuccessThreshold, "must be 1"))
  }
  allErrs = append(allErrs, validateProbe(ctr.StartupProbe, idxPath.Child("startupProbe"))...)
  // Startup-specific validation
  if ctr.StartupProbe != nil && ctr.StartupProbe.SuccessThreshold != 1 {
    allErrs = append(allErrs, field.Invalid(idxPath.Child("startupProbe", "successThreshold"), ctr.StartupProbe.SuccessThreshold, "must be 1"))
  }
```

他のProbesを含めてどのような処理フローになるか次の図をご参考ください。
![ocp4__startupprobe_process](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__startupprobe_process.png)

設定例、
```yaml
containers:
  - name: maincontainer
    startupProbe:
    httpGet:
      path: /health
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
```

設定項目の詳細説明は次のリンク先に詳しく紹介されていますので参照してください。

https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes

## 動作確認

startupProbeを利用して起動フェーズ向けの設定のみ別途指定できるので、initialDelaySecondsを利用して起動時間を予測してその時間まで監視を遅らせないで
failureThresholdを大きくして監視をスキップしないまま、定期的に初期化の完了をモニターし、実際にコンテナが起動したとたん、運用向けの他のProbesが開始できるようにより効率よく構成できます。

この構成を実際に確認してみます。Probesのチェックはkubeletから実施されるため、監視コンテナの数が多い場合や厳密に監視したい場合はkubeReservedで専用のリソースを十分設定してください。

https://docs.openshift.com/container-platform/4.5/nodes/nodes/nodes-nodes-resources-configuring.html#nodes-nodes-resources-configuring-setting_nodes-nodes-resources-configuring

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

次の通り、startupProbe,livenessProbe,readinessProbeを全て定義して、startupProbeが成功だと判定されるまで他のProbesが実際に開始されないのか確認してみましょう。
```yaml
      startupProbe:
        httpGet:
          path: /startup_healthz
          port: 8080
        failureThreshold: 20
        periodSeconds: 2
      livenessProbe:
        httpGet:
          path: /liveness_healthz
          port: 8080
        failureThreshold: 3
        periodSeconds: 3
      readinessProbe:
        httpGet:
          path: /readiness_healthz
          port: 8080
        failureThreshold: 3
        periodSeconds: 3
```

まず、startupProbeが成功するまで次の通り、livenessProbe及びreadinessProbeが一切開始されないことが分かります。
```console
Start time :  Tuesday September, 22 2020 08:27:43
10.129.2.1 - - [22/Sep/2020 08:27:44] "GET /startup_healthz HTTP/1.1" 500 -
10.129.2.1 - - [22/Sep/2020 08:27:46] "GET /startup_healthz HTTP/1.1" 500 -
:
10.129.2.1 - - [22/Sep/2020 08:28:20] "GET /startup_healthz HTTP/1.1" 500 -
10.129.2.1 - - [22/Sep/2020 08:28:22] "GET /startup_healthz HTTP/1.1" 500 -  <--- 20回目でstartupProbeが失敗され、コンテナが再起動されるまで他のProbesは一切実施されません。
```

startupProbeが失敗されてコンテナが再起動された時のイベントログの例は次の通りになります。
```console
Events:
  Type     Reason          Age                     From                                Message
  :
  Warning  Unhealthy       4m41s (x20 over 5m19s)  kubelet, worker1.ocp45.example.com  Startup probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing         3m52s                   kubelet, worker1.ocp45.example.com  Container testtools failed startup probe, will be restarted
```

startupProbeが失敗される前にアプリケーションから200を返すように設定を変更したとたん、即時にstartupProbeが成功だと判定されてその後運用向けの間隔と監視URLを設定した他のProbesが開始されることが確認できます。
```
Start time :  Tuesday September, 22 2020 08:36:03
10.129.2.1 - - [22/Sep/2020 08:36:05] "GET /startup_healthz HTTP/1.1" 500 -
10.129.2.1 - - [22/Sep/2020 08:36:07] "GET /startup_healthz HTTP/1.1" 500 -
:
10.129.2.1 - - [22/Sep/2020 08:36:37] "GET /startup_healthz HTTP/1.1" 500 -
127.0.0.1 - - [22/Sep/2020 08:36:37] "GET /?fail=false HTTP/1.1" 200 -         <--- 200ステータスコードを返すようにアプリケーションの設定を変更
10.129.2.1 - - [22/Sep/2020 08:36:39] "GET /startup_healthz HTTP/1.1" 200 -    <--- このタイミングでstartupProbeが成功だと判定
10.129.2.1 - - [22/Sep/2020 08:36:39] "GET /liveness_healthz HTTP/1.1" 200 -   <--- livenessProbeが開始
10.129.2.1 - - [22/Sep/2020 08:36:41] "GET /readiness_healthz HTTP/1.1" 200 -  <--- readinessProbeが開始
10.129.2.1 - - [22/Sep/2020 08:36:42] "GET /liveness_healthz HTTP/1.1" 200 -
10.129.2.1 - - [22/Sep/2020 08:36:44] "GET /readiness_healthz HTTP/1.1" 200 -
10.129.2.1 - - [22/Sep/2020 08:36:45] "GET /liveness_healthz HTTP/1.1" 200 -
10.129.2.1 - - [22/Sep/2020 08:36:47] "GET /readiness_healthz HTTP/1.1" 200 -
:
```

# まとめ

簡単にstartupProbeがどのように動作するかみてみました。
コンテナの起動監視が起動フェーズと運用フェーズを分けて別途指定できるようになってより適切な監視ができるようになったと感じました。
個人的には、運用するアプリケーションによっては運用の改善及び起動フェーズのみ必要なサービス間依存関係を制御する対策としても活用できるように見えまして
今後いろいろなユースケースが紹介されるのではないかと期待しています。
