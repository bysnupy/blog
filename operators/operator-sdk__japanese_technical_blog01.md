# Title: OperatorをOperator SDKで作成する on OpenShift

Red HatでOpenShiftのサポートエンジニアをしているDaein（デイン）です。

OperatorをOperator SDKで作成してOpenShiftで稼働させてみます。ドキュメント[0]のサンプルOperatorの作成のみでは情報が足りないと思っている方に、Operator作成手順をコードコメントを入れてstep by stepで解説していきます。

既にOperatorという用語に慣れている方が多いかもしれませんが、簡単に紹介します。Operatorとは管理業務を自動化する目的でKubernetes Controllerを実装するパターンで[1]、Operator SDK[2]はそのOperatorを簡単に作成できるようにしてくれる開発ツールです。

ドキュメントに記載されている手順とは別のサンプル実装として、メンテナンスページの切り替えを行うOperatorを作成してみます。このOperatorは、アプリケーションpodとメンテナンスページ用podの2つのpodをデプロイし、それら2つのpodのアクセスを切り換えます。詳細なフローとCR(CustomResource)の定義は次のようになっています。

![maintpage-operator work process](https://github.com/bysnupy/maintpage-operator/blob/master/maintpage-operator-process-diagram.png)

~~~yaml
apiVersion: maintpage.example.com/v1alpha1
kind: MaintPage
metadata:
  name: example                                    // メンテナンスページ用のPod名はCR名をベースに生成されます。例＞ example-maintpage-pod
spec:
  maintpageconfig:
    maintpagetoggle: false                         // trueにするとServiceはメンテナンスページ用のPodに振り分けするように変更されます。
    maintpageimage: quay.io/daein/maintpage:latest // メンテナンスページ用のコンテナイメージ
  appconfig:  
    appname: httpd                                 // アプリケーション用のPodをデプロイするDeploymentと関連Serviceはこの名前で作成されます。
    appimage: quay.io/daein/prodpage:latest        // アプリケーション用のコンテナイメージ
~~~

上記のCR(CustomResource)を作成するとOperatorが次のタスクを実施してくれるように作成します。

  - メンテナンスページのみ表示するPod(example-maintpage-pod)を作成する
  - アプリケーションページを表示するPod(httpd)をスケジュールするDeployment(httpd)と紐づいたService(httpd)を作成する
  - `maintpageconfig.maintpagetoggle`を利用してアプリケーションページ又はメンテナンスページへの振り分けを操作する
  - 現状どのページを表示しているか確認できる`Maintpublishstatus`項目を提供する
  - Deploymentで参照するイメージのマニュアル変更を許可しない

## Operator SDKの設置

この記事ではLinuxをベースに進めておりますが、MacOSも利用可能です。詳細は[3]を参照してください。

* Goのインストール
~~~console
# GOVERSION=1.12.9
# wget https://dl.google.com/go/go${GOVERSION}.linux-amd64.tar.gz
# tar -zxvf go${GOVERSION}.linux-amd64.tar.gz
# mkdir -p $HOME/projects/src
# vim ~/.bash_profile
export GOPATH=$HOME/projects
export GOROOT=$HOME/go
export PATH=$GOROOT/bin:$PATH
export GO111MODULE=on
# source ~/.bash_profile
~~~

* Operator SDKのインストール
~~~console
# yum install mercurial -y
# mkdir $HOME/bin
# RELEASE_VERSION=v0.10.0
# curl -OJL https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu
# mv operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu $HOME/bin/operator-sdk
# chmod u+x $HOME/bin/operator-sdk
~~~

* 動作確認
~~~console
# go version
go version go1.12.9 linux/amd64
# operator-sdk version
operator-sdk version: v0.10.0, commit: ff80b17737a6a0aade663e4827e8af3ab5a21170
~~~

以上で、インストール完了です。早速プロジェクトを作成してOperatorを作成してみましょう。

## Operatorの作成

下記で紹介するコードはロジックの実装のみ記載していてエラー及びログ処理は省略しています。

* 作業用のプロジェクトを作成: ディレクトリパスは環境に合わせて調整してください。
~~~console
# mkdir -p $GOPATH/src/github.com/bysnupy
# cd $GOPATH/src/github.com/bysnupy
# operator-sdk new maintpage-operator --repo github.com/bysnupy/maintpage-operator
# cd maintpage-operator
~~~

* CRD(Custom Resource Definition)用のAPIを追加
~~~console
# cd $GOPATH/src/github.com/bysnupy/maintpage-operator
# operator-sdk add api --api-version maintpage.example.com/v1alpha1 --kind MaintPage
~~~

* CRで設定できる設定項目を定義: 修正した内容のみ記載しています。
~~~console
# vim $GOPATH/src/github.com/bysnupy/maintpage-operator/pkg/apis/maintpage/v1alpha1/maintpage_types.go
~~~
~~~go
// CRの操作時に利用する項目パラメータを定義
type MaintPageSpec struct {
        MaintPageConfig MaintPageConfig `json:"maintpageconfig"`
        AppConfig       AppConfig       `json:"appconfig"`
}
// CRのStatusセクションの項目パラメータを定義
type MaintPageStatus struct {
        MaintPublishStatus string `json:"maintpublishstatus"`
}
...省略...
// 次は項目をまとめて定義するために追加したデータタイプで必要に応じて追加したりしてください。
type MaintPageList struct {
        metav1.TypeMeta `json:",inline"`
        metav1.ListMeta `json:"metadata,omitempty"`
        Items           []MaintPage `json:"items"`
}

type MaintPageConfig struct {
        MaintPageToggle bool   `json:"maintpagetoggle"`
        MaintPageImage  string `json:"maintpageimage"`
}

type AppConfig struct {
        AppName  string `json:"appname"`
        AppImage string `json:"appimage"`
}
~~~

* `maintpage_types.go`の内容が変更された後には忘れずに"operator-sdk generate k8s"で
  修正内容に合わせて必要なコードを再生成してください。
~~~console
# cd $GOPATH/src/github.com/bysnupy/maintpage-operator
# operator-sdk generate k8s
~~~

* 上記の手順で追加したCR(MaintPage)の定義に合わせてどのように操作するか実装するコントローラーを追加してください。
~~~console
# cd $GOPATH/src/github.com/bysnupy/maintpage-operator
# operator-sdk add controller --api-version maintpage.example.com/v1alpha1 --kind MaintPage
~~~

* 基本的に次の順で関数をカスタムしてロジックを実装します。
  - CRUDのイベントを監視(Watch)するリソースを指定します。
  - 指定したリソースにイベントが発生するたびにカスタムロジックが適用・一致(Reconcile)されます。

~~~console
# vim $GOPATH/src/github.com/bysnupy/maintpage-operator/pkg/controller/maintpage/maintpage_controller.go
~~~
~~~go
// イベントを監視したいリソースを次の関数配下で追加してください。
func add(mgr manager.Manager, r reconcile.Reconciler) error {
...省略...
        // メンテナンスページ用のPodがあるかチェックして作成するため、Podも追加します。
        err = c.Watch(&source.Kind{Type: &corev1.Pod{}}, &handler.EnqueueRequestForOwner{
                IsController: true,
                OwnerType:    &maintpagev1alpha1.MaintPage{},
        })

        // デフォルトでMaintPageは監視されますので、マニュアルでイメージが変更されることを検知するためDeploymentを次の通り追加します。
        err = c.Watch(&source.Kind{Type: &appsv1.Deployment{}}, &handler.EnqueueRequestForOwner{
                IsController: true,
                OwnerType:    &maintpagev1alpha1.MaintPage{},
        })
...省略...
// MaintPageとDeploymentでCRUDのイベントが発生した時、どのような処理をするか次の関数で実装してください。
func (r *ReconcileMaintPage) Reconcile(request reconcile.Request) (reconcile.Result, error) {
        // メンテナンス用のPodがない場合、新しく作成します。
        pod := newPodForMaintPage(maintpage)
        podfound := &corev1.Pod{}
        err = r.client.Get(context.TODO(), types.NamespacedName{Name: maintpage.Name + "-maintpage-pod", Namespace: pod.Namespace}, podfound)
        if err != nil && errors.IsNotFound(err) {
                reqLogger.Info("Creating a new Pod", "Pod.Namespace", pod.Namespace, "Pod.Name", pod.Name)
                err = r.client.Create(context.TODO(), pod)
　　　　　}
        // CRで定義したAppNameのDeploymentが存在しない場合、新しく作成します。
        depfound := &appsv1.Deployment{}
        err = r.client.Get(context.TODO(), types.NamespacedName{Name: maintpage.Spec.AppConfig.AppName, Namespace: maintpage.Namespace}, depfound)
        if err != nil && errors.IsNotFound(err) {
                dep := r.deploymentForApp(maintpage)
                err = r.client.Create(context.TODO(), dep)
　　　　　}
        // CRで定義したAppNameのServiceが存在しない場合、新しく作成します。(ServiceはWatchに追加していないため、更新は監視されません。)
        svcfound := &corev1.Service{}
        err = r.client.Get(context.TODO(), types.NamespacedName{Name: maintpage.Spec.AppConfig.AppName, Namespace: maintpage.Namespace}, svcfound)
        if err != nil && errors.IsNotFound(err) {
                // Define a new service
                svc := r.serviceForApp(maintpage)
                err = r.client.Create(context.TODO(), svc)
        }

        // CRで定義したMaintPageToggleがtrueの場合、Serviceをメンテナンスページ用のPodをEndpointとして変更してStatusセクションのMaintpublishstatusをPublishedにセットします。
        if maintpage.Spec.MaintPageConfig.MaintPageToggle {
                svcfound.Spec.Selector["app"] = maintpage.Name
                err := r.client.Update(context.TODO(), svcfound)
                statusErr := r.client.Status().Update(context.TODO(), updateMaintStatus(maintpage, "Published"))
        } else {
                if svcfound.Spec.Selector["app"] != maintpage.Spec.AppConfig.AppName {
                        svcfound.Spec.Selector["app"] = maintpage.Spec.AppConfig.AppName
                        err := r.client.Update(context.TODO(), svcfound)
                }
                // Update Status
                statusErr := r.client.Status().Update(context.TODO(), updateMaintStatus(maintpage, "Not Published"))
        }

　　　　　// Deploymentの参照イメージがCRと一致しないと元に戻します。
        if depfound.Spec.Template.Spec.Containers[0].Image != maintpage.Spec.AppConfig.AppImage {
                depfound.Spec.Template.Spec.Containers[0].Image = maintpage.Spec.AppConfig.AppImage
                err := r.client.Update(context.TODO(), depfound)
        }
~~~

コードとしては長いですが、内容はシンプルです。Watchで登録したリソースに対してイベント発生時の状態をチェックし、想定している状態に一致させるロジックを実装するだけです。
実際にDeploymentやServiceを作成するなどのCRUD操作を実装したClient APIのより詳細な情報は[4]を参照ください。

残りの関数は各リソースの定義をKubernetes APIで実装したものになります。どのように定義しているかみてみましょう。
~~~go
// メンテナンスページ用のPodをCRで定義した名前とイメージをベースに作成する関数になります。
func newPodForMaintPage(m *maintpagev1alpha1.MaintPage) *corev1.Pod {
        maintpageimage := m.Spec.MaintPageConfig.MaintPageImage
        labels := map[string]string{
                "app": m.Name,
        }
        return &corev1.Pod{
                ObjectMeta: metav1.ObjectMeta{
                        Name:      m.Name + "-maintpage-pod",
                        Namespace: m.Namespace,
                        Labels:    labels,
                },
                Spec: corev1.PodSpec{
                        Containers: []corev1.Container{
                                {
                                        Name:    "maintpage",
                                        Image:   maintpageimage,
                                },
                        },
                },
        }
}

// アプリケーション用のPodをCRで定義した名前とイメージをベースにデプロイする関数になります。
func (r *ReconcileMaintPage) deploymentForApp(m *maintpagev1alpha1.MaintPage) *appsv1.Deployment {
        appname  := m.Spec.AppConfig.AppName
        appimage := m.Spec.AppConfig.AppImage
        replicas := int32(1)

        dep := &appsv1.Deployment{
                ObjectMeta: metav1.ObjectMeta{
                        Name:      appname,
                        Namespace: m.Namespace,
                },
                Spec: appsv1.DeploymentSpec{
                        Replicas: &replicas,
                        Selector: &metav1.LabelSelector{
                                MatchLabels: map[string]string{"app": appname},
                        },
                        Template: corev1.PodTemplateSpec{
                                ObjectMeta: metav1.ObjectMeta{
                                        Labels: map[string]string{"app": appname},
                                },
                                Spec: corev1.PodSpec{
                                        Containers: []corev1.Container{{
                                                Image:   appimage,
                                                Name:    appname,
                                        }},
                                },
                        },
                },
        }
        // Set MaintPage instance as the owner and controller
        controllerutil.SetControllerReference(m, dep, r.scheme)
        return dep
}

// CRで定義した名前でServiceを作成する関数になります。
func (r *ReconcileMaintPage) serviceForApp(m *maintpagev1alpha1.MaintPage) *corev1.Service {
        appname  := m.Spec.AppConfig.AppName

        svc := &corev1.Service{
                ObjectMeta: metav1.ObjectMeta{
                        Name:      appname,
                        Namespace: m.Namespace,
                },
                Spec: corev1.ServiceSpec{
                        Ports: []corev1.ServicePort{{
                                Name:       "8080-tcp",
                                Protocol:   "TCP",
                                Port:       8080,
                                TargetPort: intstr.FromInt(8080),
                        }},
                        Selector: map[string]string{"app": appname},
                },
        }
        // Set MaintPage instance as the owner and controller
        controllerutil.SetControllerReference(m, svc, r.scheme)
        return svc
}

// StatusセクションのMaintPublishStatus項目を渡された引数に合わせてセットする関数です。
func updateMaintStatus(m *maintpagev1alpha1.MaintPage, status string) *maintpagev1alpha1.MaintPage {
        m.Status.MaintPublishStatus = status
        return m
}
~~~

コードの全文は[7]から確認できます。

## Operatorのビルド及び設置

次の手順でOperatorをビルドしてイメージを適切なレジストリにpushしてください。また、メンテナンスページ及びアプリケーションページのコンテナイメージは別途用意してください。こちらではhttpdコンテナにそれぞれ"Maintenance Page !"と"Production Page !"が記載されたindex.htmlを追加したものでテストしています。

* Operatorのビルド
~~~console
# oc create -f $GOPATH/src/github.com/bysnupy/maintpage-operator/deploy/maintpage_v1alpha1_maintpage_crd.yaml
# cd $GOPATH/src/github.com/bysnupy/maintpage-operator
# go mod vendor
# operator-sdk build quay.io/daein/maintpage-operator:v0.0.1
# sed -i 's|REPLACE_IMAGE|quay.io/daein/maintpage-operator:v0.0.1|g' deploy/operator.yaml
# docker push quay.io/daein/maintpage-operator:v0.0.1
~~~

* Operatorの設置: プロジェクト名は適切にご調整ください。設置前にpushしたレジストリからイメージpullできるか確認してください。
~~~console
# oc new-project maintpage-operator
# oc create -f deploy/service_account.yaml
# oc create -f deploy/role.yaml
# oc create -f deploy/role_binding.yaml
# oc create -f deploy/operator.yaml

# oc get pod -n maintpage-operator
NAME                                  READY     STATUS    RESTARTS   AGE
maintpage-operator-6df9d9c85c-s7xmw   1/1       Running   0          20s
~~~

* CR(MaintPage)を定義してメンテナンス及びアプリケーションページ用のPodを作成してください。
~~~console
# oc create -n maintpage-operator -f - <<EOF
apiVersion: maintpage.example.com/v1alpha1
kind: MaintPage
metadata:
  name: example
spec:
  maintpageconfig:
    maintpagetoggle: false
    maintpageimage: quay.io/daein/maintpage:latest
  appconfig:  
    appname: httpd
    appimage: quay.io/daein/prodpage:latest
EOF

# oc get pod -n maintpage-operator
NAME                                  READY     STATUS    RESTARTS   AGE
example-maintpage-pod                 1/1       Running   0          20s
httpd-784b46459b-spfzg                1/1       Running   0          19s
maintpage-operator-6df9d9c85c-s7xmw   1/1       Running   0          40s

# oc get svc httpd -n maintpage-operator
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
httpd     ClusterIP   172.30.131.106   <none>        8080/TCP   21m
~~~

## Operatorの動作チェック

CRの`Maintpagetoggle`をtrueに更新してメンテナンスページの切り替えとDeploymentのイメージをマニュアルで更新しても変更されないか確認しましょう。

次の通り、Deploymentのイメージはマニュアルで変更しても更新されないことが確認できます。
~~~console
# oc edit deployment/httpd -n maintpage-operator
...
spec:
  containers:
  - image: quay.io/daein/prodpage:notfound

# oc get deployment/httpd -o yaml | grep image:
      - image: quay.io/daein/prodpage:latest
~~~

次の通り、CR(MaintPage)の`Maintpagetoggle`をtrue/falseに更新することでページ表示が変わることが確認できます。
~~~console
// 変更前の状態は"Maintpublishstatus:  Not Published"です。
# oc describe maintpage example -n maintpage-operator
Name:         example
...
Status:
  Maintpublishstatus:  Not Published
Events:                <none>

// 別のターミナルでService当てにリクエストし続けてページの変更をモニターしましょう。
# while :; do echo "$(date '+%H:%M:%S'): $(curl -s http://httpd.maintpage-operator.svc.cluster.local:8080)" ;sleep 1; done
23:55:17: Production Page !
23:55:18: Production Page !
23:55:19: Production Page !
...

// "maintpageconfig.maintpagetoggle: true"に更新してください。
# oc edit maintpage example -n maintpage-operator
...
spec:
  ...
  maintpageconfig:
    ...
    maintpagetoggle: true

// 表示ページが"Production Page !"から"Maintenance Page !"に変更されることが確認できます。
# while :; do echo "$(date '+%H:%M:%S'): $(curl -s http://httpd.maintpage-operator.svc.cluster.local:8080)" ;sleep 1; done
...
23:55:57: Production Page !
23:55:58: Production Page !
23:55:59: Maintenance Page !
23:56:01: Maintenance Page !
23:56:02: Maintenance Page !

// 更新した後の状態は"Maintpublishstatus: 　Published"になりました。
# oc describe maintpage example
Name:         example
...
Status:
  Maintpublishstatus:  Published
Events:                <none>
~~~

"maintpagetoggle: false"に戻して元の状態になるかも確認してください。

本記事では任意で決めた仕様に合わせてOperator SDKを利用してどのようにOperatorが作成されるかみてみました。こちらで紹介している内容以外でも実装仕様によってはfinalizer[5]やOLM(Operator Lifecycle Manager)[6]の実装も検討する必要があるでしょう。既に数多くのOperatorがOperatorHub.io[8]とGitHub[9]で提供されてOpenSourceとして公開されていますので実装に困った時には参考にすると良いと思います。

- [0] Building a Go-based Memcached Operator using the Operator SDK
  -  [https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-getting-started.html#building-memcached-operator-using-osdk_osdk-getting-started]
- [1] Understanding Operators
  -  [https://docs.openshift.com/container-platform/4.1/applications/operators/olm-what-operators-are.html]
- [2] Getting started with the Operator SDK
  -  [https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-getting-started.html#osdk-architecture_osdk-getting-started]
- [3] Installing the Operator SDK CLI
  -  [https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-getting-started.html#osdk-installing-cli_osdk-getting-started]
- [4] Operator-SDK: Controller Runtime Client API
　-　[https://github.com/operator-framework/operator-sdk/blob/master/doc/user/client.md]
- [5] Handle Cleanup on Deletion
  -  [https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md#handle-cleanup-on-deletion]
- [6] Managing a Memcached Operator using the Operator Lifecycle Manager
  -  [https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-getting-started.html#managing-memcached-operator-using-olm_osdk-getting-started]
- [7] Maintpage-Operator Github
　-　[https://github.com/bysnupy/maintpage-operator]
- [8] OperatorHub.io
  - [https://operatorhub.io/]
- [9] Awesome Operators in the Wild
  - [https://github.com/operator-framework/awesome-operators#awesome-operators-in-the-wild]
