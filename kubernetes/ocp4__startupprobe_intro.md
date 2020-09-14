# startupProbeでアプリケーションの初期化を監視する on OpenShift

## Summary

Red HatでOpenShiftのサポートエンジニアをしているDaein（デイン）です。
Kubernetes 1.18リリースでstartupProbeがBeta featureとして昇格され、デフォルトで有効になりました。
OpenShift 4.5ではこのKubernetes 1.18が採用されているのでこの機能を使ってみたいと思います。

## Probesとは

ProbesというのはKubernetesでPodの中で起動しているコンテナを監視する機能で今までLivenss ProbeとReadiness Probeがそれぞれ違う目的で提供していました。

Probes name | Description
-|-
Liveness Probe | コンテナが動いているかを示して、失敗するとrestartPolicyに従って対象コンテナを再起動させる
Readiness Probe | コンテナがServiceのリクエストを受けることができるかを示して、失敗するとEndpointsコントローラーが全てのServiceからそのPod IPを削除して外部からアクセスできなくなります。

新しく追加されたstartupProbeはコンテナ内のアプリケーションが起動したかどうかを起動時のみ評価するもので、失敗するとrestartPolicyに従って対象コンテナを再起動させます。
Liveness/Readiness ProbesはこのstartupProbeが設定された場合はこのProbeが成功するまで開始されないので起動時間が長くかかるアプリケーションの起動監視のみ別途設定したい時に活用できます。


![ocp4_probes_timeline](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4_probes_timeline.png)
