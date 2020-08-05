# Security Context Constrains(SCC)の適用プロセス on OpenShift

Red HatでOpenShiftのサポートエンジニアをしているDaein（デイン）です。

Security Context Constraints(SCC)を正しく設定するためにどのようなプロセスで適用されるか説明します。
簡単にSecurity Context Constraints(SCC)について紹介すると、OpenShiftでPodが実行できるアクション及びアクセスを制御するリソースになります。
通常対象Podを作成する認証ユーザーやリンクされているServiceAccountに許可されたSCCを利用してPodを起動させて制御できます。詳細情報は次のドキュメントをご参照ください。

https://docs.openshift.com/container-platform/4.5/authentication/managing-security-context-constraints.html#admission_configuring-internal-oauth

# 処理の流れ

"Priority"の項目を一番高く指定したSCCのみ適用されると考えしやすいですが、実際には対象Podの設定によって左右されます。
また、SCCを複数付与してもその機能や権限が合わせて適用されるわけではないのでビルトインのSCCで対応できない場合は別途カスタムSCCを作成する必要があります。
処理フローをシンプルに説明すると次の通りになります。

1. 操作する認証ユーザー又はPodを起動させるServiceAccount(SA)に許可されたSCCを洗い出します。
1. そのSCCを高いPriority順にソートします。
2. ソートされたSCC順にPodの設定を満たしているかチェックします。
3. 最初マッチされたSCCがあったらその1つSCCでPodは起動されます。この処理順序は"Priority"設定に影響を受けます。
4. マッチされたSCCがなかった場合はPodは無効になって起動されません。

![scc_process_flow](https://github.com/bysnupy/blog/blob/master/kubernetes/scc-process.png)

# 動作確認

上記の動作を実際にOCP4.5の環境で次の確認してみましょう。
"default" ServiceAccount(SA)に"anyuid"、"hostnetwork" 2つのSCCを追加して"hostPath"をPodに設定した前後の違いを確認してみましょう。
"hostNetwork: true"はアサインされたSCCの中で"hostnetwork"のみ提供できるため、"anyuid"がより高い"Priority"が設定されていてもPodに"hostNetwork: true"がリクエストされた場合は"hostnetwork" SCCでPodが起動されます。
また、"cluster-admin"のクラスタロールを持つ認証ユーザーであればSCCを別途設定しなくてもどのSCCでも利用できるため、"Forbidden"エラーなしで適切なSCCが設定されてPodが作成されることも確認します。

デフォルトで提供しているSCCの一覧は次の通りになります。cluster-admin権限を持つアカウントは次のSCCが全てご利用できます。
```cmd
$ oc get scc
NAME               PRIV    CAPS         SELINUX     RUNASUSER          FSGROUP     SUPGROUP    PRIORITY     READONLYROOTFS   VOLUMES
anyuid             false   <no value>   MustRunAs   RunAsAny           RunAsAny    RunAsAny    10           false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
hostaccess         false   <no value>   MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <no value>   false            [configMap downwardAPI emptyDir hostPath persistentVolumeClaim projected secret]
hostmount-anyuid   false   <no value>   MustRunAs   RunAsAny           RunAsAny    RunAsAny    <no value>   false            [configMap downwardAPI emptyDir hostPath nfs persistentVolumeClaim projected secret]
hostnetwork        false   <no value>   MustRunAs   MustRunAsRange     MustRunAs   MustRunAs   <no value>   false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
node-exporter      true    <no value>   RunAsAny    RunAsAny           RunAsAny    RunAsAny    <no value>   false            [*]
nonroot            false   <no value>   MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny    <no value>   false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
privileged         true    [*]          RunAsAny    RunAsAny           RunAsAny    RunAsAny    <no value>   false            [*]
restricted         false   <no value>   MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <no value>   false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
```

テストのため、プロジェクトを作成し、その配下のdefault SAにanyuidとhostnetworkの追加しておきます。
```cmd
$ oc new-project test-scc
$ oc adm policy add-scc-to-user anyuid     -z default
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "default"

$ oc adm policy add-scc-to-user hostnetwork -z default
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:hostnetwork added: "default"
```

一般ユーザーとして認証して"hostNetwork: true"を設定しないPodと設定したPodを作成して起動させます。
```cmd
$ oc login -u normal-user -p YOURPASSWORD
$ oc whoami
normal-user

$ oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-anyuid
spec:
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    image: registry.redhat.io/ubi8/ubi
    name: test
---
apiVersion: v1
kind: Pod
metadata:
  name: test-hostnetwork
spec:
  hostNetwork: true
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    image: registry.redhat.io/ubi8/ubi
    name: test
EOF
```

次の通り、同じdefault SAで起動したPodでも"hostNetwork: true"が設定されたPodはanyuidの"Priority"がより高く設定されているにも関わらず、"houstnetwork" SCCが適用されていることが確認できます。
```cmd
$ oc get pod
NAME               READY   STATUS    RESTARTS   AGE
test-anyuid        1/1     Running   0          3m22s
test-hostnetwork   1/1     Running   0          3m22s

$ oc get pod -o yaml | grep -E '^    name:|openshift.io/scc:|serviceAccountName:'
      openshift.io/scc: anyuid
    name: test-anyuid
    serviceAccountName: default
		
      openshift.io/scc: hostnetwork
    name: test-hostnetwork
    serviceAccountName: default
```

Podに設定された機能が利用可能なSCCの中でマッチされなかった場合は次のForbiddenエラーメッセージが表示され、Podが作成できなくなります。
```cmd
Error from server (Forbidden): error when creating "STDIN": pods "test-hostnetwork" is forbidden: unable to validate against any security context constraint: [provider anyuid: .spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.containers[0].securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used provider restricted: .spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.containers[0].securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used]
```

比較として、"cluster-admin"のクラスタロールを持つ認証ユーザーであればただPodを作成するだけでSCCが適切に設定されますので、テスト時には注意が必要です。
```cmd
$ oc new-project test-scc-adminuser
$ oc login -u admin-user -p YOURPASSWORD
$ oc whoami
admin-user

$ oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-anyuid
spec:
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    image: registry.redhat.io/ubi8/ubi
    name: test
---
apiVersion: v1
kind: Pod
metadata:
  name: test-hostnetwork
spec:
  hostNetwork: true
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    image: registry.redhat.io/ubi8/ubi
    name: test
EOF
pod/test-anyuid created
pod/test-hostnetwork created

$ oc get pod -o yaml | grep -E '^    name:|openshift.io/scc:|serviceAccountName:'
      openshift.io/scc: anyuid
    name: test-anyuid
    serviceAccountName: default
    
      openshift.io/scc: hostnetwork
    name: test-hostnetwork
    serviceAccountName: default
```

# 実装からの確認

主に"computeSecurityContext"関数からSCCの適用プロセスが実施されていますので興味ある方はご参考ください。ソースコード全文はこちらをご参照ください。
https://github.com/openshift/apiserver-library-go/blob/a7bc13e3e6504ddc7056ab9a5d2d6c61f7eb45b9/pkg/securitycontextconstraints/sccadmission/admission.go#L133-L237

```golang
func (c *constraint) computeSecurityContext(ctx context.Context, a admission.Attributes, pod *coreapi.Pod, specMutationAllowed bool, validatedSCCHint string) (*coreapi.Pod, string, field.ErrorList, error) {
	// get all constraints that are usable by the user
  :
  // "FindApplicableSCCs"関すから許可されたSCCを"Priority"順にソートした一覧が返却されます。
  constraints, err := sccmatching.NewDefaultSCCMatcher(c.sccLister, nil).FindApplicableSCCs(ctx, a.GetNamespace())
  :
  // こちらからPod設定にマッチするSCCを確認するループが実装されています。
  loop:
  	for _, provider := range providers {
  		// Get the SCC attributes required to decide whether the SCC applies for current user/SA
  		sccName := provider.GetSCCName()
  		sccUsers := provider.GetSCCUsers()
  		sccGroups := provider.GetSCCGroups()

  		// continue to the next provider if the current SCC one does not apply to either the user or the serviceaccount
  		if !sccmatching.ConstraintAppliesTo(ctx, sccName, sccUsers, sccGroups, userInfo, a.GetNamespace(), c.authorizer) &&
  			!(saUserInfo != nil && sccmatching.ConstraintAppliesTo(ctx, sccName, sccUsers, sccGroups, saUserInfo, a.GetNamespace(), c.authorizer)) {
  			continue
  		}
      :
      // the entire pod validated
      switch {
      case specMutationAllowed:
        // if mutation is allowed, use the first found SCC that allows the pod.
        // This behavior is different from Kubernetes which tries to search a non-mutating provider
        // even on creating. We prefer most restrictive SCC in this case even if it mutates a pod.
:  
```

# まとめ

以上、簡単にSCCがどのように処理されて適用されるか紹介しました。SCCの設計関連の資料も補足としてご参考頂ければと思います。

https://github.com/openshift/origin/blob/bce0020d05343434cb9f6453ea62ca78ee82f064/docs/proposals/security-context-constraints.md#admission

通常専用のSAを作成し、必要最低限のSCCを利用してPodを運用するべきですが、11つのSAで仕様が違う複数のPodを運用したりする場合は参考にしてください。

