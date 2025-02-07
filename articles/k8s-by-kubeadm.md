---
title: "kubeadmでk8sクラスタを構築する"
emoji: "🦅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Kubernetes, kubeadm]
published: true
---

## Kubernetes

Kubernetesとは、複数のコンピュータでコンテナをいい感じに動かしてくれるものです。
Kubernetesの説明はいろんなサイトに書いてあるため、そちらを参照してください。
公式サイトも参考になります。

https://kubernetes.io/docs/concepts/overview/

## kubeadm

[kubeadm](https://github.com/kubernetes/kubeadm)は、Kubernetesクラスタを構築するためのツールの１つです。
他にも、[kops](https://github.com/kubernetes/kops)や[kubespray](https://github.com/kubernetes-sigs/kubespray)などがありますが、kubeadmは最小限の構成でクラスタを構築することができます。

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/

## 環境

今回は、研究室にあった3台のミニPCを使って、進めていきます。
それぞれの機器のスペックは以下の通りです。

|ノード|ホスト名|種類|CPU|メモリ|ストレージ|OS|カーネル|
|-----|---|---|---|-----|--------|--|-------|
|マスター|NucBox7|[NucBox7](https://manuals.plus/gmktec/nucbox-7-7s-mini-pc-manual)|4コア|16GB|512GB|Ubuntu 24.04|6.8.0-49-generic|
|ワーカー|bmax04|[BMAX](https://www.bmaxit.com/MaxMini-B1-Plus-pd751447098.html)|2コア|6GB|64GB|Ubuntu 24.04|6.8.0-49-generic|
|ワーカー|bmax05|[BMAX](https://www.bmaxit.com/MaxMini-B1-Plus-pd751447098.html)|2コア|6GB|64GB|Ubuntu 24.04|6.8.0-49-generic|

## Requirement

kubeadmでクラスタを構築するにあたり、最低条件は以下になります。

- CPU: 2コア
- メモリ: 2GB(2GBの場合アプリ用のスペースはほとんどない)
- ネットワーク: クラスター内のすべてのマシン間で通信できる
- ユニークな hostname, MACアドレス, product_uuid
- マシン内の特定のポートが開いている
- Swapがオフ

Ubuntuでは、デフォルトでSwapがオンになっているため、 */etc/fstab* を編集して、swapをオフにします。

## ツールのインストール

まず、Kubernetesクラスタに必要なツールをインストールします。

### コンテナランタイムのインストール

カーネルパラメーターで、IPフォワーディングを有効にします。これにより、ホストOSのパケットをコンテナに転送することができます。

```bash
$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
$ sudo sysctl --system # 設定の反映
```

[containerd](https://github.com/containerd/containerd)を全てのノードにインストールします。

```bash
$ sudo apt update
$ sudo apt install containerd

$ sudo bash -c "containerd config default > /etc/containerd/config.toml"
$ sudo vim /etc/containerd/config.toml
  disabled_plugins = [] # criが含まれていない
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true # falseからtrueに書き換える
$ systemctl restart containerd
$ systemctl status containerd
```

### kubeadm, kubelet, kubectlのインストール

[kubeadm](https://github.com/kubernetes/kubeadm), [kubelet](https://github.com/kubernetes/kubelet), [kubectl](https://github.com/kubernetes/kubectl)を全てのノードにインストールします。

```bash
＄ sudo apt-get update
＄ sudo apt-get install -y apt-transport-https ca-certificates curl gpg

$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```

## クラスタの構築

準備ができたら、クラスタを構築します。

### コントロールプレーンの構築

まず、マスターノードでコントロールプレーンを構築します。
kubeadm initコマンドを実行するだけで完了します。

```bash
$ sudo kubeadm config images pull # 確認
$ sudo kubeadm init
Your Kubernetes control-plane has initialized successfully!
```

kubeadm initの出力結果と同じように、kubeconfigファイルを作成します。

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### ノードの追加

次に、ワーカーノードを追加します。

```bash
$ sudo kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

ここまでで、CoreDNSを除くコンポーネントを起動することができました。
CNIをベースとするネットワークプラグインをインストールする前には、CoreDNSは起動しません。

### ネットワークプラグインのインストール

ネットワークプラグインは[いくつかあります](https://github.com/containernetworking/cni?tab=readme-ov-file#3rd-party-plugins)が、今回は[Cilium](https://github.com/cilium/cilium)を使います。
まず、マスターノード上でCilium CLIをインストールします。

```bash
$ CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
$ if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
$ sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
$ sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
$ rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

その後、Cilium CLIでCilium Podをインストールします。

```bash
$ cilium install --version 1.16.4
$ cilium status # 確認
```

`kube-system namespace` のPodを確認して、全てRunningになっていたら、成功です。

```bash
$ kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
cilium-7v4jt                       1/1     Running   0          21m
cilium-bb5s2                       1/1     Running   0          21m
cilium-envoy-d6mz8                 1/1     Running   0          21m
cilium-envoy-dstdf                 1/1     Running   0          21m
cilium-envoy-rh9cg                 1/1     Running   0          21m
cilium-gzhq5                       1/1     Running   0          21m
cilium-operator-6946d85fd9-gcqr8   1/1     Running   0          21m
coredns-668d6bf9bc-c28sk           1/1     Running   0          23m
coredns-668d6bf9bc-chn7p           1/1     Running   0          23m
etcd-nucbox7                       1/1     Running   10         23m
kube-apiserver-nucbox7             1/1     Running   10         23m
kube-controller-manager-nucbox7    1/1     Running   7          23m
kube-proxy-27cgh                   1/1     Running   0          23m
kube-proxy-jr9fb                   1/1     Running   0          22m
kube-proxy-rwc67                   1/1     Running   0          22m
kube-scheduler-nucbox7             1/1     Running   11         23m

$ cilium connectivity test
```

## kube-proxyの排除

Ciliumは、eBPFを使うことで、kube-proxy+iptablesによるルーティングを置き換えることができます。

```bash
$ kubectl delete daemonsets.apps -n kube-system kube-proxy
$ kubectl delete configmaps -n kube-system kube-proxy
```

すべてのノードでコマンドを実行する。

```bash
sudo bash -c "iptables-save | grep -v KUBE | iptables-restore"
```

`kubeProxyReplacement`と`nodeIPAM.enabled`を有効にして、Ciliumを再インストールします。

```bash
$ helm repo add cilium https://helm.cilium.io/
$ helm install cilium cilium/cilium --version 1.16.4 \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=10.70.70.27 \
    --set k8sServicePort=6443
    --set nodeIPAM.enabled=true
```

## サンプルアプリのデプロイ

サンプルとして用意されているGuestBookアプリケーションをデプロイしてみます。

```bash
$ kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-deployment.yaml
$ kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-service.yaml
$ kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-deployment.yaml
$ kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-service.yaml
$ kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml
```

ただ、Cilium Node IPAM LBを有効にしているため、ServiceのタイプをLoadBalancerにして、frontendにアクセスできるようにします。
*spec.type*をLoadBalancerに変更して、*spec.loadBalancerClass*に`io.cilium/node`を指定して、Serviceを作成します。

```bash
$ wget https://k8s.io/examples/application/guestbook/frontend-service.yaml
$ vim frontend-service.yaml # 変更
$ kubectl apply -f frontend-service.yaml
```

これにより、ワーカーノードのIPアドレス+ポート番号で、サービスにアクセスできるようになります。

https://kubernetes.io/docs/tutorials/stateless-application/guestbook/

https://sreake.com/blog/cilium-node-ipam-lb-load-balancing/

## ユーザアカウントの作成

今のままだと、kubeconfigファイルを使って、クラスタにアクセスできるのはマスターノードだけです。
マスターノードの`$HOME/.kube/config`をコピーすれば、管理用PCからもアクセスできますが、監査ログには全てマスターノードのアクセスとして記録されてしまいます。
そのため、ユーザアカウントを作成して、管理用PCからは、それを使ってアクセスするようにします。

まず、秘密鍵を生成して、証明書署名要求を作成します。その後、証明書署名要求をBase64でエンコードします。

```bash
$ openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 > user-name.pem # 秘密鍵の生成
$ openssl req -new -key user-name.pem -out kobayashi-s.csr -subj "/CN=user-name" # 証明書署名要求の生成
$ cat user-name.csr | base64 | tr -d '\n' # 証明書署名要求のBase64エンコード
```

CSRのマニフェストを作成して、ユーザアカウントの作成をリクエストします。

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user-request-[user-name]
spec:
  signerName: kubernetes.io/kube-apiserver-client
  groups:
  - system:authenticated
  request: [Base64エンコードした証明書署名要求]
  usages:
  - digital signature
  - key encipherment
  - client auth
```

既存のアカウントでCSRを承認します。

```bash
$ kubectl apply -f user-request.yaml
$ kubectl get certificatesigningrequests.certificates.k8s.io # Pendingを確認
$ kubectl certificate approve user-request-[user-name] # 承認
$ kubectl get certificatesigningrequests.certificates.k8s.io # Approvedを確認
```

証明書を取得して、kubeconfigファイルを作成します。これは、管理PCで実行します。

```bash
$ kubectl get csr user-request-kobayashi-s -o jsonpath='{.status.certificate}' | base64 -d > [user-name].crt
$ kubectl config set-cluster lab-cluster --insecure-skip-tls-verify=true --server=https://10.70.70.27:6443
$ kubectl config set-credentials kobayashi-s --client-certificate=kobayashi-s.crt --client-key=kobayashi-s.pem --embed-certs=true
$ kubectl config set-context lab-cluster --cluster=lab-cluster --user=kobayashi-s
$ kubectl config use-context lab-cluster
```

ここまでで、ユーザアカウントの作成が完了しました。
しかし、まだクラスタにアクセスする権限がありません。
ClusterRoleBindingを作成して、ユーザアカウントに権限を付与します。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: link-[user-name]-to-[role-name]
subjects:
- kind: User
  name: [user-name]
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: [role-name]
  apiGroup: rbac.authorization.k8s.io
```

:::details 管理者権限を付与する例

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: link-[user-name]-to-cluster-admin
subjects:
- kind: User
  name: [user-name]
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

:::

```bash
$ kubectl apply -f cluster-role-binding.yaml
$ kubectl get clusterrolebindings.rbac.authorization.k8s.io # 確認
```

これ以降は管理PCから、Kubernetesクラスタを操作することができます。

## ダッシュボードの作成

何も動かしていないと寂しいので、ダッシュボードを作成します。
Kubernetesダッシュボードは、KubernetesクラスタをWebブラウザで操作するためのツールです。

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

helmを使って、ダッシュボードをインストールします。

```bash
$ helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
$ helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace k
ubernetes-dashboard
```

ダッシュボードにアクセスするために、ServiceAccountとClusterRoleBindingを作成します。

```bash
$ kubectl apply -f service-account.yaml
$ kubectl apply -f cluster-role-binding.yaml
$ kubectl -n kubernetes-dashboard create token admin-user # Bearerトークンの発行
$ kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

<https://localhost:8443> にアクセスして、このようなダッシュボードが見れることが確認できます。
![dashboard.png](https://storage.googleapis.com/zenn-user-upload/7d71c149fbb3-20241217.png)
