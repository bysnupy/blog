Title: ビルド・デプロイの安定的な処理時間の維持 on OpenShift

Red HatのDaein(デイン)です。OpenShift(Kubernetes)環境でビルド及デプロイの性能劣化を抑える方法について紹介します。
OpenShiftでは次の通りソースコードをビルドして新しいイメージを作成し、そのイメージを用いて新しいPodをデプロイしています。

![build_deploy_process](https://github.com/bysnupy/blog/blob/master/kubernetes/k8s__build_deploy_resources.png)

ビルドとデプロイ時間に影響する項目としては次のものが考えられます。

処理時間|担当先
-|-
Podのスケジュールから作成までの時間|OpenShift(Kubernetes)
ビルドしたイメージをPUSH及PULLする時間|Image Registry
build Podが稼働されて実際にアプリケーションをビルドする処理時間|アプリケーションのビルド要件に依存
デプロイされたアプリケーションのPodが起動完了するまでの時間|プリケーションの稼働要件に依存
