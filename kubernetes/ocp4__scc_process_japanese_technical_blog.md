# Security Context Constrains(SCC)の適用プロセス on OpenShift

Red HatでOpenShiftのサポートエンジニアをしているDaein（デイン）です。

Security Context Constraints(SCC)を正しく設定するためにどのようなプロセスで適用されていか確認していきます。

Security Context Constraints(SCC)は、OpenShiftでPodが実行できるアクション及びアクセスを制御するリソースです。Podを作成するユーザーやリンクされているServiceAccountで許可されたSCCの制限を利用し、Podを起動させます。詳細情報は次のドキュメントをご参照ください。

https://docs.openshift.com/container-platform/4.5/authentication/managing-security-context-constraints.html#admission_configuring-internal-oauth

# 処理の流れ

まず、"Priority"の項目を一番高く指定したSCCのみ適用されると考えてしまいがちですが、実際には対象Podの設定によっても左右されます。
また、SCCを複数付与してもその機能や権限が合わせて適用されるわけではないのでビルトインのSCCで対応できない場合は別途カスタムSCCを作成する必要があります。
処理の流れを簡単に説明すると、次のようになります。

1. 操作するユーザーまたはPodで参照されるServiceAccount(SA)で許可されたSCCを洗い出します。
2. 洗い出されたSCCをPriorityの高い順にソートします。
3. ソートされたSCC順でPodの設定に適用できるSCCがあるかチェックします。
4. 最初にマッチした1つのSCCでPodを作成します。この処理順序は、"Priority"設定が影響します。（"Priority"が指定あれていない場合は"0"としてみなされます。）
   - 同じ"Priority"の場合はより制約されたSCCが優先されます。
   - "Priority"と"制約"も同じの場合はSCC名のソート順で優先されます。
5. マッチするSCCがなかった場合はPodは無効になって作成されません。

![scc_process_flow](https://github.com/bysnupy/blog/blob/master/kubernetes/scc-process.png)

# 動作確認

上記の動作を実際にOCP4.5の環境で次の通り確認してみましょう。

## "Priority"よりPodの設定が優先されるパターン

"default" ServiceAccount(SA)に"anyuid"、"hostnetwork" 2つのSCCをアサインして"hostNetwork: true"をPodに設定した前後の違いを確認してみましょう。
"hostNetwork: true"はアサインされたSCCの中で"hostnetwork"のみ提供できるため、"anyuid"がより高い"Priority"が設定されていてもPodに"hostNetwork: true"がリクエストされた場合は"hostnetwork" SCCでPodが作成されます。


テストのため、プロジェクトを作成し、その配下のdefault SAにanyuidとhostnetworkのアサインします。
```cmd
$ oc new-project test-scc
$ oc adm policy add-scc-to-user anyuid     -z default -n test-scc
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "default"

$ oc adm policy add-scc-to-user hostnetwork -z default -n test-scc
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:hostnetwork added: "default"
```

一般ユーザーとして認証して"hostNetwork: true"を設定しないPodと設定したPodを作成します。
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

Podに設定された機能や権限が提供可能なものがアサインされたSCCの中でマッチされなかった場合は次の通りForbiddenエラーメッセージが表示され、Podが作成できなくなります。
```cmd
Error from server (Forbidden): error when creating "STDIN": pods "test-hostnetwork" is forbidden: unable to validate against any security context constraint: [provider anyuid: .spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.containers[0].securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used provider restricted: .spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.containers[0].securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used]
```

## 同じ"Priority"の場合はより強い制約のSCCが適用されるパターン

"default" ServiceAccount(SA)に"root"が指定できない制限をしたまま、DockerfileのUSERに指定された特定のUIDでPodを作成する意図として"nonroot" SCCをアサインしているにも関わらず、デフォルトSCCの"restricted"が同じ"Priority"でかつより強い制約を持っているため優先して起用されます。"nonroot" SCCを適用するためには明示的に"securityContext.runAsUser: XXX"で特定のUIDをPod設定に指定するか、"nonroot" SCCの"Priority"を"restricted"より高く設定する必要があります。Pod設定での適用は上記のパターンで確認しましたので今回は"Priority"を修正するパターンで確認します。

テストのため、プロジェクトを作成し、その配下のdefault SAにnonrootのみアサインします。
```cmd
$ oc new-project test-scc2
$ oc adm policy add-scc-to-user nonroot     -z default -n test-scc2
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:nonroot added: "default"
```

一般ユーザーとして認証してPodを作成します。
```cmd
$ oc login -u normal-user -p YOURPASSWORD
$ oc whoami
normal-user

$ oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-nonroot
spec:
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    image: registry.redhat.io/rhel8/nginx-116
    name: test
```

次の通り、同じ"Priority"のSCCであってもより制約が強い"restricted" SCCが適用されることが確認できます。

```cmd
$ oc get pod
NAME                     READY   STATUS    RESTARTS   AGE
test-nonroot             1/1     Running   0          16m

$ oc rsh test-nonroot id
uid=1000600000(1000600000) gid=0(root) groups=0(root),1000600000

$ oc get pod test-nonroot -o yaml | grep -E 'openshift.io/scc:|serviceAccountName:'
    openshift.io/scc: restricted
  serviceAccountName: default
```

続いて"cluster-admin"のクラスタロールを持つアカウントで"nonroot" SCCの"Priority"を"20"に修正してから一般ユーザーとして認証してPodを作成します。
```
$ oc login -u admin-user -p YOURPASSWORD
$ oc whoami
admin-user
$ oc edit scc nonroot
:
priority: 20
:

$ oc get scc restricted nonroot
NAME         PRIV    CAPS         SELINUX     RUNASUSER          FSGROUP     SUPGROUP   PRIORITY     READONLYROOTFS   VOLUMES
restricted   false   <no value>   MustRunAs   MustRunAsRange     MustRunAs   RunAsAny   <no value>   false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
nonroot      false   <no value>   MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny   20           false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]

$ oc login -u normal-user -p YOURPASSWORD
$ oc whoami
normal-user

$ oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-nonroot-after-prio
spec:
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    image: registry.redhat.io/rhel8/nginx-116
    name: test
EOF
```

次の通り、より高い"Priority"が設定された"nonroot" SCCが適用され、DockerfileのUSERに指定されたUIDが暗黙的に反映されていることが確認できます。
```cmd
$ oc get pod
NAME                      READY   STATUS    RESTARTS   AGE
test-nonroot              1/1     Running   0          89m
test-nonroot-after-prio   1/1     Running   0          62s

$ oc rsh test-nonroot-after-prio id
uid=1001(default) gid=0(root) groups=0(root)

$ oc get pod test-nonroot-after-prio -o yaml | grep -E 'openshift.io/scc:|serviceAccountName:'
    openshift.io/scc: nonroot
  serviceAccountName: default
```

## "cluster-admin"のクラスタロールで操作した場合、別途SCCのアサインなしで関連機能が利用できるパターン

"cluster-admin"のクラスタロールを持つ認証ユーザーであればSCCを別途設定しなくてもどのSCCでも利用できるため、"Forbidden"エラーなしで適切なSCCが設定されてPodが作成されることを確認します。

デフォルトで提供しているSCCの一覧は次の通りになります。cluster-admin権限を持つアカウントは次のSCCが利用できます。
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

一般ユーザーと比較して、"cluster-admin"のクラスタロールを持つ認証ユーザーであれば、別途アサイン作業なしでただPodを作成するだけでSCCが設定されますので、一般ユーザーに設定をテストする場合は注意してください。
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

# 実装の確認

以下は、興味ある方はご参考ください。
主に"computeSecurityContext"関数からSCCの適用プロセスが実施されます。ソースコード全文はこちらで参照できます。
https://github.com/openshift/apiserver-library-go/blob/a7bc13e3e6504ddc7056ab9a5d2d6c61f7eb45b9/pkg/securitycontextconstraints/sccadmission/admission.go#L133-L237

```golang
func (c *constraint) computeSecurityContext(ctx context.Context, a admission.Attributes, pod *coreapi.Pod, specMutationAllowed bool, validatedSCCHint string) (*coreapi.Pod, string, field.ErrorList, error) {
	// get all constraints that are usable by the user
  :
  // "FindApplicableSCCs"関数から許可されたSCCを"Priority"順にソートした一覧が返却されます。
  constraints, err := sccmatching.NewDefaultSCCMatcher(c.sccLister, nil).FindApplicableSCCs(ctx, a.GetNamespace())
  :
  // ここからPod設定にマッチするSCCを確認するループが実装されています。
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

簡単にSCCがどのように処理されて適用されるか紹介しました。SCCの設計関連の資料も補足としてご参考にしてください。

https://github.com/openshift/origin/blob/bce0020d05343434cb9f6453ea62ca78ee82f064/docs/proposals/security-context-constraints.md#admission

通常は、専用のSAを作成し、必要最低限のSCCを利用してPodを運用するべきですが、1つのSAで仕様が違う複数のPodを運用する場合には注意が必要です。
