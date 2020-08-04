# Security Context Constrains(SCC)のPriorityについて

Red HatでOpenShiftのサポートエンジニアをしているDaein（デイン）です。

高い"Priority"をSCCに指定しても他のSCCが利用されるパターンがありますが、その時"Priority"が適用されるSCCを選定する処理でどのように動作するか説明します。
簡単にSecurity Context Constraints(SCC)について紹介すると、OpenShiftでPodが実行できるアクション及びアクセスを制御するリソースになります。
通常対象Podに関連付けられているServiceAccountに必要な権限や機能が許可されたSCCを付与することでPodの動作が制御できます。詳細情報は次のドキュメントをご参照ください。

https://docs.openshift.com/container-platform/4.5/authentication/managing-security-context-constraints.html#admission_configuring-internal-oauth

# "Priority"の動作

"Priority"の項目を一番高く指定したSCCのみ適用されると考えしやすいですが、実際には対象Podの設定によって左右されます。
処理フローをシンプルに説明すると次の通りになります。

1. 利用可能なSCCを高いPriority順にソートします。
2. ソートされたSCC順にPodの設定を満たしているかチェックします。
3. マッチされたSCCがあったらそのSCCでPodは起動されます。
4. マッチされたSCCがなかった場合はPodは無効になって起動されません。

# 動作チェック

上記の動作を実際にOCP4.5の環境で確認してみましょう。
"default" ServiceAccountに"anyuid"、"hostnetwork" 2つのSCCを追加して"hostPath"をPodに設定した前後の違いを確認してみましょう。
"hostNetwork: true"はアサインされたSCCの中で"hostnetwork"のみ提供しているため、"anyuid"がより高い"Priority"が設定されてもPodに"hostPath"が設定された場合は"hostaccess" SCCでPodが起動されます。

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
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:hostaccess added: "default"
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

Podに設定された機能が利用可能なSCCの中でマッチされなかった場合は次のエラーメッセージが表示され、Podが作成できなくなります。
```cmd
Error from server (Forbidden): error when creating "STDIN": pods "test-hostnetwork" is forbidden: unable to validate against any security context constraint: [provider anyuid: .spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.containers[0].securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used provider restricted: .spec.securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used spec.containers[0].securityContext.hostNetwork: Invalid value: true: Host network is not allowed to be used]
```

# まとめ

Podごとに専用ServiceAccount(SA)を作成して必要最低限の権限や機能をSCCを利用して制御するユースケースが望ましいですが、
同じSAに複数のSCCを付与して複数のPodをそれぞれの設定で運用したい場合はこの記事が参考になると思います。
