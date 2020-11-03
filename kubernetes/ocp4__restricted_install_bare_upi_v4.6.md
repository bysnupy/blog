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
  * DNSレコード
  * Load Balancerの構成
  * Mirrorレジストリの構成
* インストールの実施
* インストールの完了

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
worker1.ocp46rt.priv.local | RHCOS | 192.168.9.35 | 16 | 16 GB | 150GB | Workerノード
worker2.ocp46rt.priv.local | RHCOS | 192.168.9.36 | 16 | 16 GB | 150GB | Workerノード
worker3.ocp46rt.priv.local | RHCOS | 192.168.9.37 | 16 | 16 GB | 150GB | Workerノード

### DNSレコード

こちらではOpenShiftがインストールされるネットワーク制限環境とそれ以外のネットワークでDNSがそれぞれ存在していて次の通り各ネームサーバに登録するレコードは次の通りになります。
ネットワーク制限環境で外部ネットワークを経由しないでoc CLIを利用するため、apiやingressのVIPレコードを重複して登録しています。

ネットワーク制限環境用のDNSに登録が必要なレコード
ホスト名 | IPs  |  備考
--------|------|-------
mirror.reg.priv.local | 192.168.9.50 | 
api.ocp46rt.priv.local | 192.168.9.10 | API load balancerの内部向けVIP
api-int.ocp46rt.priv.local | 192.168.9.10 | API load balancerの内部向けVIP
*.apps.ocp46rt.example.com | 192.168.9.10 | Ingress load balancerの内部向けVIP
bootstrap.ocp46rt.priv.local | 192.168.9.31 | 
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

### Load Balancerの構成

次の構成に合わせて踏み台ホスト（bastion）にHAProxyを利用して構築します。

API Load Balancer

![restricted_network_lb_api](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__restricted_network_api_lb.png)

Application Ingress Load Balancer

![restricted_network_lb_ingress](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__restricted_network_ingress_lb.png)

詳細はドキュメントを参照してください。

Networking requirements for user-provisioned infrastructure
  https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#installation-network-user-infra_installing-restricted-networks-bare-metal
  
踏み台ホスト(bastion)にHAProxyをインストールして次の通り設定します。
```console
# API for External
frontend external-lb-for-api-ocp46rt
  bind    10.0.1.10:6443
  option  tcplog
  mode    tcp
  default_backend  backend-api-6443-ocp46rt

# API for Ingernal
frontend internal-lb-for-api-ocp46rt
  bind    192.168.9.10:6443
  option  tcplog
  mode    tcp
  default_backend  backend-api-6443-ocp46rt

frontend internal-lb-for-machineconfig-ocp46rt
  bind    192.168.9.10:22623
  option  tcplog
  mode    tcp
  default_backend  backend-machineconfig-22623-ocp46rt

# Ingress for External
frontend external-lb-for-ingressrouter-HTTP-ocp46rt
  bind    10.0.1.10:80
  option  tcplog
  mode    tcp
  default_backend  backend-ingressrouter-80-ocp46rt

frontend external-lb-for-ingressrouter-HTTPS-ocp46rt
  bind    10.0.1.10:443
  option  tcplog
  mode    tcp
  default_backend  backend-ingressrouter-443-ocp46rt

# Ingress for Internal
frontend internal-lb-for-ingressrouter-HTTP-ocp46rt
  bind    192.168.9.10:80
  option  tcplog
  mode    tcp
  default_backend  backend-ingressrouter-80-ocp46rt

frontend internal-lb-for-ingressrouter-HTTPS-ocp46rt
  bind    192.168.9.10:443
  option  tcplog
  mode    tcp
  default_backend  backend-ingressrouter-443-ocp46rt

# Master node pool for API
backend backend-api-6443-ocp46rt
  mode     tcp
  balance  roundrobin
  option   ssl-hello-chk
  server   bootstrap  bootstrap.ocp46rt.priv.local:6443 check
  server   master1    master1.ocp46rt.priv.local:6443   check
  server   master2    master2.ocp46rt.priv.local:6443   check
  server   master3    master3.ocp46rt.priv.local:6443   check

# Master node pool for machineconfig
backend backend-machineconfig-22623-ocp46rt
  mode     tcp
  balance  roundrobin
  option   ssl-hello-chk
  server   bootstrap  bootstrap.ocp46rt.priv.local:22623 check
  server   master1    master1.ocp46rt.priv.local:22623   check
  server   master2    master2.ocp46rt.priv.local:22623   check
  server   master3    master3.ocp46rt.priv.local:22623   check

# Worker node pool for Ingress router
backend backend-ingressrouter-80-ocp46rt
  mode     tcp
  balance  roundrobin
  server   worker1    worker1.ocp46rt.priv.local:80  check
  server   worker2    worker2.ocp46rt.priv.local:80  check
  server   worker3    worker3.ocp46rt.priv.local:80  check

backend backend-ingressrouter-443-ocp46rt
  mode     tcp
  balance  roundrobin
  option   ssl-hello-chk
  server   worker1    worker1.ocp46rt.priv.local:443 check
  server   worker2    worker2.ocp46rt.priv.local:443 check
  server   worker3    worker3.ocp46rt.priv.local:443 check
```

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

```console
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

```console
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

最後に次のコマンドで"openshift-install"インストーラーバイナリにミラーレジストリに固定させたものとしてビルドしたものを取得する手順が残っていますが、こちらはquay.ioにアクセスしようとする既知の不具合の影響でv4.6.1では実施しないで、代りにインストールするOpenShiftと同じバージョンのopenshift-installバイナリを利用してください。

```console
$ GODEBUG=x509ignoreCN=0 oc adm -a ${LOCAL_SECRET_JSON} release extract \
  --command=openshift-install \
  "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}"
error: unable to read image quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:...: unauthorized: access to the requested resource is not authorized
```

詳細内容はこちらのバグレポートをご参考ください。
oc adm release extract --command, --tools doesn't pull from localregistry when given a localregistry/image 
  https://bugzilla.redhat.com/show_bug.cgi?id=1823143

これでインストールに必要なミラーレジストリが用意できました。次はOpenShift 4.6をインストールしてみましょう。

## インストールの実施

こちらの作業は踏み台ホスト(bastion)で全て実施されます。

SSHキーペアの準備はドキュメントを参照して事前に作成しておきます。

Generating an SSH private key and adding it to the agent
  https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#ssh-agent-using_installing-restricted-networks-bare-metal

インストール設定ファイルの手動作成してその中にミラーレジストリのCA証明書、SSHキーとイメージをダウンロードする時取得した"imageContentSources"のミラーレジストリーの情報を追加します。


```yaml
user2@bastion ~$ mkdir install_dir
user2@bastion ~$ cat <<EOF > install_dir/install-config.yaml
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled   
  name: worker
  replicas: 0 
controlPlane:
  hyperthreading: Enabled   
  name: master 
  replicas: 3 
metadata:
  name: ocp46rt
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OpenShiftSDN
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
fips: false 
pullSecret: '{"auths":{"mirror.reg.priv.local:5000":{"auth":...}}}'
sshKey: 'ssh-rsa ...'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - mirror.reg.priv.local:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - mirror.reg.priv.local:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
EOF
```

デフォルトでユーザーPodのスケジュールがMasterノードにも有効になっていますが、control plane系の安定稼動のためこちらを無効にします。

```console
user2@bastion ~$ openshift-install version
openshift-install 4.6.1
:

user2@bastion ~$ openshift-install create manifests --dir install_dir
INFO Consuming Install Config from target directory 
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings 
INFO Manifests created in: install_dir/manifests and install_dir/openshift

user2@bastion ~$ sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' install_dir/manifests/cluster-scheduler-02-config.yml 
user2@bastion ~$ cat install_dir/manifests/cluster-scheduler-02-config.yml | grep mastersSchedulable
  mastersSchedulable: false
```

ノードホスト起動時にネットワーク経由でIgnitionファイルが取得できるようIgnition生成する前にweb serverも設定しておきます。
Web server(httpd)をインストールしてポートを8080にして次の通りDocumentRootを設定します。

```console
/var/www/html/ocp46rt/
├── ign
    ├── bootstrap.ign (owner: apache, group: apache, 0644)
    ├── master.ign    (owner: apache, group: apache, 0644)
    └── worker.ign    (owner: apache, group: apache, 0644)
```

Ignitionファイルの生成
```console
user2@bastion ~$ openshift-install create ignition-configs --dir install_dir
INFO Consuming Worker Machines from target directory 
INFO Consuming Master Machines from target directory 
INFO Consuming Common Manifests from target directory 
INFO Consuming Openshift Manifests from target directory 
INFO Consuming OpenShift Install (Manifests) from target directory 
INFO Ignition-Configs created in: install_dir and install_dir/auth

user2@bastion ~$ cp install_dir/*.ign /var/www/html/ocp46rt/ign/
user2@bastion ~$ sudo chown apache:apache -R /var/www/html/ocp46rt/*
```

"rhcos-4.6.1-x86_64-live.x86_64.iso"を持ってBootstrapノードホストを先に起動させてから、続いてMasterノードホスト1〜3号機を同じISOで起動してから
コンソールを通して必要なネットワーク設定を実施した後、coreos-installerを利用してネットワーク及びIgnition設定を適用して再起動するとBootStrapが初期化完了した次第Masterノードも初期化されます。
直接カーネルパラメータを利用してRHCOSをインストールする方法も有効なので、そちらの詳細は次のドキュメントを見てください。

Advanced RHCOS installation reference
  https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#installation-user-infra-machines-static-network_installing-restricted-networks-bare-metal

自分の環境ではDHCPがネットワークで既に設定されていて、カーネルパラメータでネットワーク設定しないと"--copy-network"を利用して固定IPが正しく設定できなかったので"--append-karg"を利用してネットワーク設定していますが、
"--copy-network"オプションは"nmcli"で設定した内容をRHCOSに投入してくれますので、より自由にネットワークが設定できると思います。ご利用環境に合わせてお試しください。

```console
$ sudo nmcli con mod "Wired Connection" ipv4.method manual \
  ipv4.addresses 192.168.9.xx/24 ipv4.gateway 192.168.9.10 \
  ipv4.dns 192.168.9.10
$ sudo nmcli con down "Wired Connection"
$ sudo nmcli con up   "Wired Connection"
$ sudo nmcli general hostname master1.ocp46rt.priv.local
$ sudo coreos-installer install /dev/sda \
  --ignition-url=http://192.168.9.10:8080/ocp46rt/ign/master.ign \
  --insecure-ignition \
  --append-karg "ip=192.168.9.xx::192.168.9.10:255.255.255.0:HOSTNAME.ocp45rt.priv.local:ens3:none nameserver=192.168.9.10" 
```

Masterノード設定例、
![rhcos_coreos_installer_example](https://github.com/bysnupy/blog/blob/master/kubernetes/ocp4__rhcos_coreos_installer_config_example.png)

Bootstrap及び全Masterノードを起動してからbootstrapプロセスが完了するまで待機します。

```console
user2@bastion ~$ openshift-install wait-for bootstrap-complete --dir install_dir --log-level debug
DEBUG OpenShift Installer 4.6.1                    
DEBUG Built from commit ebdbda57fc18d3b73e69f0f2cc499ddfca7e6593 
INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp46rt.example.com:6443... 
:
INFO API v1.19.0+d59ce34 up                       
INFO Waiting up to 30m0s for bootstrapping to complete... 
:
INFO It is now safe to remove the bootstrap resources 
DEBUG Time elapsed per stage:                      
DEBUG Bootstrap Complete: 7m58s                    
DEBUG                API: 2m52s                    
INFO Time elapsed: 7m58s   
```

無事完了したらインストールしたクラスタに認証してWorkerノードを追加します。

```console
user2@bastion ~$ export KUBECONFIG=/home/user2/install_dir/auth/kubeconfig

user2@bastion ~$ oc get node
NAME                         STATUS   ROLES    AGE    VERSION
master1.ocp46rt.priv.local   Ready    master   122m   v1.19.0+d59ce34
master2.ocp46rt.priv.local   Ready    master   64m    v1.19.0+d59ce34
master3.ocp46rt.priv.local   Ready    master   17m    v1.19.0+d59ce34
```

Masterノードと同じ手順でWorkerノードの初期設定を実施します。暫く時間が経つとWorkerノードからのCSR(クライアントとサーバ証明書)が一覧されますので承認してWorkerノードをReadyにして追加します。

```console
user2@bastion ~$ oc get csr | grep Pending
csr-8zz9h   5m31s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-ltb9x   8m23s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-nh6v6   75s     kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending

user2@bastion ~$ oc get csr -o name | xargs oc adm certificate approve
:

user2@bastion ~$ oc get node
NAME                         STATUS     ROLES    AGE    VERSION
master1.ocp46rt.priv.local   Ready      master   166m   v1.19.0+d59ce34
master2.ocp46rt.priv.local   Ready      master   108m   v1.19.0+d59ce34
master3.ocp46rt.priv.local   Ready      master   61m    v1.19.0+d59ce34
worker1.ocp46rt.priv.local   NotReady   worker   104s   v1.19.0+d59ce34
worker2.ocp46rt.priv.local   NotReady   worker   0s     v1.19.0+d59ce34
worker3.ocp46rt.priv.local   NotReady   worker   106s   v1.19.0+d59ce34

user2@bastion ~$ oc get csr | grep Pending
csr-4c8fp   2m26s   kubernetes.io/kubelet-serving                 system:node:worker3.ocp46rt.priv.local                                      Pending
csr-dvpdl   2m25s   kubernetes.io/kubelet-serving                 system:node:worker1.ocp46rt.priv.local                                      Pending
csr-rd5lr   40s     kubernetes.io/kubelet-serving                 system:node:worker2.ocp46rt.priv.local                                      Pending

user2@bastion ~$ oc get csr -o name | xargs oc adm certificate approve
:

user2@bastion ~$ oc get node
NAME                         STATUS   ROLES    AGE     VERSION
master1.ocp46rt.priv.local   Ready    master   168m    v1.19.0+d59ce34
master2.ocp46rt.priv.local   Ready    master   110m    v1.19.0+d59ce34
master3.ocp46rt.priv.local   Ready    master   63m     v1.19.0+d59ce34
worker1.ocp46rt.priv.local   Ready    worker   3m37s   v1.19.0+d59ce34
worker2.ocp46rt.priv.local   Ready    worker   112s    v1.19.0+d59ce34
worker3.ocp46rt.priv.local   Ready    worker   3m39s   v1.19.0+d59ce34
```

Image Registryを"managementState":"Managed"にして適切なストレージを設定して有効にします。こちらではemptyDirに設定していますが、その他のストレージの設定は次のドキュメントを参照してください。

Image registry storage configuration
  https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html#installation-registry-storage-config_installing-restricted-networks-bare-metal

```
user2@bastion ~$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
user2@bastion ~$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
```

全てのClusterOperatorが"AVAILABLE: True"になっているか確認した後、インストールを完了させます。

```console
user2@bastion ~$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.6.1     True        False         False      89s
cloud-credential                           4.6.1     True        False         False      4h49m
cluster-autoscaler                         4.6.1     True        False         False      179m
config-operator                            4.6.1     True        False         False      3h
console                                    4.6.1     True        False         False      64s
csi-snapshot-controller                    4.6.1     True        False         False      179m
dns                                        4.6.1     True        False         False      178m
etcd                                       4.6.1     True        False         False      75m
image-registry                             4.6.1     True        False         False      70m
ingress                                    4.6.1     True        False         False      2m59s
insights                                   4.6.1     True        False         False      3h
kube-apiserver                             4.6.1     True        False         False      73m
kube-controller-manager                    4.6.1     True        False         False      178m
kube-scheduler                             4.6.1     True        False         False      177m
kube-storage-version-migrator              4.6.1     True        False         False      177m
machine-api                                4.6.1     True        False         False      179m
machine-approver                           4.6.1     True        False         False      179m
machine-config                             4.6.1     True        False         False      75m
marketplace                                4.6.1     True        False         False      178m
monitoring                                 4.6.1     True        False         False      69m
network                                    4.6.1     True        False         False      3h
node-tuning                                4.6.1     True        False         False      3h
openshift-apiserver                        4.6.1     True        False         False      70m
openshift-controller-manager               4.6.1     True        False         False      177m
openshift-samples                          4.6.1     True        False         False      59m
operator-lifecycle-manager                 4.6.1     True        False         False      179m
operator-lifecycle-manager-catalog         4.6.1     True        False         False      179m
operator-lifecycle-manager-packageserver   4.6.1     True        False         False      71m
service-ca                                 4.6.1     True        False         False      3h
storage                                    4.6.1     True        False         False      3h
```

## インストールの完了

```console
user2@bastion ~$ openshift-install wait-for install-complete --log-level debug
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/clusters/ocp46rt/install_dir/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp46rt.example.com 
INFO Login to the console with user: "kubeadmin", and password: "..." 
INFO Time elapsed: 0s 
```

これでOpenShift 4.6 Bare-metal UPIでのインストールが完了しました。その後、Post-Installの設定があると思いますが、機会があればその設定も紹介したいと思います。
