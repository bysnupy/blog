# OpenShift 4.6をBare-metal UPIでネットワーク制限環境下でインストールする

## 概要
Extended Update Support(EUS)が提供されるOpenShift 4.6が多くの改善と共にリリースされました。
今回はインターネットにアクセスできない環境配下でOpenShift 4.6をBare-metal UPIインストール方法で構築してみた内容を紹介します。

  OpenShift Container Platform 4.6 release notes
    https://docs.openshift.com/container-platform/4.6/release_notes/ocp-4-6-release-notes.html
  Red Hat OpenShift Extended Update Support (EUS) Overview
    https://access.redhat.com/support/policy/updates/openshift-eus

章立ては次の通りになります。

* インストール構成の概要
* インストールの事前準備
  * ノード及び作業ホスト構成
  * Load Balancerの構成
  * DNSレコード
  * Mirrorレジストリの構成
* インストールの実施

## インストール構成の概要

基本的にドキュメントで紹介されている手順でOpenShiftのインストールは進みますが、事前準備で別途用意した他のシステムはこちらで任意で決めて設定及び構成しています。

Installing a cluster on bare metal in a restricted network
  https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html

全体的な構成は次のイメージの通りになります。

![restricted_network_network_diagram](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__restricted_network_ocp_diagram.png)

### ノード及び作業ホスト構成

ホスト名 | OS | IP | vCPU |メモリ | DISK | 備考 
--------|-----|-----|----|------|------|------
image-down.lan.example.com | RHEL8 | 10.0.1.50 | 2 | 1GB | 30GB | 作業ホスト、インターネットから必要なイメージを1次にダウンロードして踏み台に転送させるホスト
bastion.priv.example.com | RHEL8 | 10.0.1.10, 192.168.9.10 | 2 | 4GB | 50GB | 作業ホスト、インストーラーを実行するホスト、Load Balancer(HAProxy)とインストールに必要なコンテンツを提供するWeb Server(httpd)も一緒に構成
mirror.reg.priv.local | RHEL8 | 192.168.9.50 | 4 | 8GB | 1TB | Mirrorレジストリ、docker v2 APIが対応できるイメージレジストリ用のホスト、インストール及びOperator用のイメージがこちらに保存
bootstrap.ocp46rt.priv.local | RHCOS | 192.168.9.31 | 8 | 16GB | 50GB | Masterノードを初期化まで必要なホスト
master1.ocp46rt.priv.local | RHCOS | 192.168.9.32 | 8 | 16GB | 100GB | Masterノード
master2.ocp46rt.priv.local | RHCOS | 192.168.9.33 | 8 | 16GB | 100GB | Masterノード
master3.ocp46rt.priv.local | RHCOS | 192.168.9.34 | 8 | 16GB | 100GB | Masterノード
worker1.ocp46rt.priv.local | RHCOS | 192.168.9.35 | 16 | 16 GB | 50GB + 100GB | Workerノード
worker2.ocp46rt.priv.local | RHCOS | 192.168.9.36 | 16 | 16 GB | 50GB + 100GB | Workerノード
worker3.ocp46rt.priv.local | RHCOS | 192.168.9.37 | 16 | 16 GB | 50GB + 100GB | Workerノード


### Load Balancerの構成

次の構成に合わせて踏み台ホスト（bastion）にHAProxyを利用して構築します。

API Load Balancer
![restricted_network_lb_api](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__restricted_network_api_lb.png)

Application Ingress Load Balancer
![restricted_network_lb_ingress](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__restricted_network_ingress_lb.png)

詳細はドキュメントを参照してください。

Networking requirements for user-provisioned infrastructure
  https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#installation-network-user-infra_installing-restricted-networks-bare-metal

### DNSレコード

こちらではOpenShiftがインストールされるネットワーク制限環境とそれ以外のネットワークでDNSがそれぞれ存在していて次の通り各ネームサーバに登録するレコードは次の通りになります。
ネットワーク制限環境で外部ネットワークを経由しないでoc CLIを利用するため、apiやingressのVIPレコードを重複して登録しています。

ネットワーク制限環境用のDNSに登録が必要なレコード
ホスト名 | IPs  |  備考
--------|------|-------
bootstrap.ocp46rt.priv.local | 192.168.9.31 | 
mirror.reg.priv.local | 192.168.9.50 | 
api.ocp46rt.priv.local | 192.168.9.10 | API load balancerの内部向けVIP
api-int.ocp46rt.priv.local | 192.168.9.10 | API load balancerの内部向けVIP
*.apps.ocp46rt.example.com | 192.168.9.10 | Ingress load balancerの内部向けVIP
master1.ocp46rt.priv.local | 192.168.9.32 | 
master2.ocp46rt.priv.local | 192.168.9.33 | 
master3.ocp46rt.priv.local | 192.168.9.34 |  
worker1.ocp46rt.priv.local | 192.168.9.35 | 
worker2.ocp46rt.priv.local | 192.168.9.36 | 
worker3.ocp46rt.priv.local | 192.168.9.37 | 

その他のネットワークからLBを経由してOpenShiftにアクセスするために必要なDNSレコード
ホスト名 | IPs  |  備考
--------|------|-------
api.ocp46rt.priv.local | 10.0.1.10 | API load balancerの外向けVIP
*.apps.ocp46rt.example.com | 10.0.1.10 | Ingress load balancerの外向けVIP

### Mirrorレジストリの構成

#### インストール用のイメージをダウンロード及び踏み台ホストに転送

まず、インストールするOpenShiftと同じバージョンのoc CLIバイナリが必要です。インストール方法は次のドキュメントを参照してください。

Installing the CLI on Linux
  https://docs.openshift.com/container-platform/4.6/installing/install_config/installing-restricted-networks-preparations.html#cli-installing-cli-on-linux_installing-restricted-networks-preparations

この作業はインターネットにアクセスできる"image-down"ホストで実施されます。

```console
user1@image-down ~$ dig +short quay.io
54.210.241.179
52.0.92.170
52.201.153.168
:
user1@image-down ~$ oc version
Client Version: 4.6.1
```

"cloud.redhat.com"から取得したpull-secretを次の通りファイルとして保存します。
また、ダウンロードしたイメージが保存されるディレクトリを作成し、oc CLIで参照される引数を環境変数として事前に定義しておきます。

```console
user1@image-down ~$ cat <<EOF > pull-secret-for-RH-registries.json
{"auths":{"cloud.openshift.com":{"auth":...}}}
EOF

user1@image-down ~$ mkdir ./copied_images

user1@image-down ~$ cat <<EOF > source_file_for_copied_images
export OCP_RELEASE=4.6.1
export LOCAL_REGISTRY=mirror.reg.priv.local:5000
export LOCAL_REPOSITORY=ocp4/openshift4
export PRODUCT_REPO=openshift-release-dev
export LOCAL_SECRET_JSON=/home/user1/pull-secret-for-RH-registries.json
export RELEASE_NAME=ocp-release
export ARCHITECTURE=x86_64
export REMOVABLE_MEDIA_PATH=/home/user1/copied_images
EOF

user1@image-down ~$ source source_file_for_copied_images
```

"--dry-run"で実際にミラーを実施しないで"install-config.yaml"に必要な"imageContentSources"を先に取得してメモしてください。
```console
user1@image-down ~$ oc adm -a ${LOCAL_SECRET_JSON} release mirror \
                       --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
                       --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
                       --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --dry-run
info: Mirroring 120 images to mirror.reg.priv.local:5000/ocp4/openshift4 ...
:
Success
Update image:  mirror.reg.priv.local:5000/ocp4/openshift4:4.6.1-x86_64
Mirror prefix: mirror.reg.priv.local:5000/ocp4/openshift4

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - mirror.reg.priv.local:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - mirror.reg.priv.local:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - mirror.reg.priv.local:5000/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - mirror.reg.priv.local:5000/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev

user1@image-down ~$
```

先程作成したディレクトリにインストールに必要なイメージをダウンロードします。

```
user1@image-down ~$ oc adm release mirror -a ${LOCAL_SECRET_JSON} \
                       --to-dir=${REMOVABLE_MEDIA_PATH}/mirror \
                       quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}
info: Mirroring 120 images to file://openshift/release ...
:
info: Mirroring completed in 25m33.42s (4.552MB/s)

Success
Update image:  openshift/release:4.6.1

To upload local images to a registry, run:

    oc image mirror --from-dir=/home/user1/copied_images/mirror 'file://openshift/release:4.6.1*' REGISTRY/REPOSITORY

Configmap signature file /home/user1/copied_images/mirror/config/signature-sha256-d78292e9730dd387.yaml created

user1@image-down ~$ du -hs ./copied_images/
6.6G	./copied_images/
```

ダウンロードしたイメージは内部でシンボリックリンクが利用されているため、tarでアーカイブするか"rsync -a"などでその構成を維持して踏み台ホストに転送してください。

```
user1@image-down ~$ rsync -avc ./copied_images/* user2@bastion:copied_images_bastion/
:
sent 6,981,691,453 bytes  received 9,832 bytes  58,424,278.54 bytes/sec
total size is 6,979,915,019  speedup is 1.00
```

#### 転送されたイメージをミラーレジストリにアップロード

まず、インストールするOpenShiftと同じバージョンのoc CLIバイナリが必要です。インストール方法は次のドキュメントを参照してください。

Installing the CLI on Linux
  https://docs.openshift.com/container-platform/4.6/installing/install_config/installing-restricted-networks-preparations.html#cli-installing-cli-on-linux_installing-restricted-networks-preparations

この作業はインターネットにアクセスできない"bastion"ホストで実施されます。

```console
user1@bastion ~$ dig +short mirror.reg.priv.local
192.168.9.50

user1@bastion ~$ oc version
Client Version: 4.6.1
```

イメージをミラーレジストリにアップロードするために必要なJSON形式の認証ファイルの作成とoc CLIで参照される引数を環境変数として事前に定義しておきます。

```console
user2@bastion ~$ cat <<EOF > pull-secret-for-MIRROR-registries.json
{"auths":{"mirror.reg.priv.local:5000":{"auth":...}}}
EOF

user2@bastion ~$ cat <<EOF > source_file_for_upload_images
export OCP_RELEASE=4.6.1
export LOCAL_REGISTRY=mirror.reg.priv.local:5000
export LOCAL_REPOSITORY=ocp4/openshift4
export PRODUCT_REPO=openshift-release-dev
export LOCAL_SECRET_JSON=/home/user2/pull-secret-for-MIRROR-registries.json  <--- こちらが先程と違います。
export RELEASE_NAME=ocp-release
export ARCHITECTURE=x86_64
export REMOVABLE_MEDIA_PATH=/home/user2/copied_images_bastion                 <--- こちらが先程と違います。
EOF

user2@bastion ~$ source ./source_file_for_upload_images
```

転送されたイメージをミラーレジストリにアップロードします。

Go言語のバージョンによる証明書関連の仕様変更によって次のメッセージでアップロードできない場合がありますが、"GODEBUG=x509ignoreCN=0"を指定して再実行してください。

```console
x509: certificate relies on legacy Common Name field, use SANs or temporarily enable Common Name matching with GODEBUG=x509ignoreCN=0
```

詳細はこちらのバグレポートをご参考ください。
Need to document TLS CommonName deprecation 
  https://bugzilla.redhat.com/show_bug.cgi?id=1886892

```console
user2@bastion ~$ GODEBUG=x509ignoreCN=0 oc image mirror -a ${LOCAL_SECRET_JSON} \
                 --from-dir=${REMOVABLE_MEDIA_PATH}/mirror \
                 "file://openshift/release:${OCP_RELEASE}*" \
                 ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}
mirror.reg.priv.local:5000/
:
info: Mirroring completed in 1m10.82s (97.37MB/s)
```

これでインストールに必要なミラーレジストリが用意できました。次はOpenShift 4.6をインストールしてみましょう。
