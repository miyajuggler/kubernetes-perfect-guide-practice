## よくわからん単語

- ハイパーバイザー

-

-

## コマンド

```sh
$ kubectl config get-clusters
CURRENT   NAME                                                           CLUSTER                                                        AUTHINFO                                                       NAMESPACE
          arn:aws:eks:ap-northeast-1:403128494076:cluster/test-cluster   arn:aws:eks:ap-northeast-1:403128494076:cluster/test-cluster   arn:aws:eks:ap-northeast-1:403128494076:cluster/test-cluster
*         docker-desktop                                                 docker-desktop                                                 docker-desktop

$ kubectl config current-context                                              ○ test-cluster
arn:aws:eks:ap-northeast-1:403128494076:cluster/test-cluster

$ kubectl config use-context docker-desktop                                   ○ test-cluster
Switched to context "docker-desktop".

$ kubectl config current-context                                            ○ docker-desktop
docker-desktop
```

## kind 導入

```sh
$ brew install kind

$ kind version
kind v0.12.0 go1.17.8 darwin/amd64
```

クラスタの構成は yaml ファイルで記述する。(kind.yaml を参照)

```
$ kind create cluster --config kind.yaml --name kindcluster
Creating cluster "kindcluster" ...
 ✓ Ensuring node image (kindest/node:v1.18.2) 🖼
 ✗ Preparing nodes 📦 📦 📦 📦 📦 📦
ERROR: failed to create cluster: could not find a log line that matches "Reached target .*Multi-User System.*|detected cgroup v1"
```

エラーになった。
いろいろ調べた感じ image がうまく取れてない気がするので書き換える。

```diff
- image: kindest/node:v1.18.2
+ image: kindest/node@sha256:719e6e188ea1dc13e5ffabca854117d55daab7930351839632dea158f81fdc4c
# + image: kindest/node:v1.18.20@sha256:719e6e188ea1dc13e5ffabca854117d55daab7930351839632dea158f81fdc4c
```

```
$ kind create cluster --config kind.yaml --name kindcluster
Creating cluster "kindcluster" ...
 ✓ Ensuring node image (kindest/node) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Configuring the external load balancer ⚖️
 ✓ Writing configuration 📜
 ✗ Starting control-plane 🕹️
ERROR: failed to create cluster: failed to init node with kubeadm: command "docker exec --privileged kindcluster-control-plane kubeadm init --skip-phases=preflight --config=/kind/kubeadm.conf --skip-token-print --v=6" failed with error: exit status 137
Command Output: I0325 14:45:05.292199     206 initconfiguration.go:200] loading configuration from "/kind/kubeadm.conf"
[config] WARNING: Ignored YAML document with GroupVersionKind kubeadm.k8s.io/v1beta2, Kind=JoinConfiguration
W0325 14:45:05.435974     206 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.20
I0325 14:45:05.448659     206 kubelet.go:64] Stopping the kubelet
[kubelet-start] WARNING: unable to stop the kubelet service momentarily: [exit status 1]
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[kubelet-start] WARNING: unable to start the kubelet service: [failed to reload systemd: exit status 1]
[kubelet-start] Please ensure kubelet is reloaded and running manually.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
I0325 14:45:05.778729     206 certs.go:103] creating a new certificate authority for ca
```

なんのこっちゃ。。。

アプローチを変えてみて、[これ](https://qiita.com/kyontra/items/b1696df6ea072fa48c34)を参考にやってみた

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```sh
kind create cluster --config kind.yaml --name kindcluster
Creating cluster "kindcluster" ...
 ✓ Ensuring node image (kindest/node:v1.23.4) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kindcluster"
You can now use your cluster with:

kubectl cluster-info --context kind-kindcluster

Have a nice day! 👋
$ kubectl get no                                              ○ kind-kindcluster
NAME                        STATUS     ROLES                  AGE   VERSION
kindcluster-control-plane   Ready      control-plane,master   46s   v1.23.4
kindcluster-worker          Ready      <none>                 13s   v1.23.4
kindcluster-worker2         Ready      <none>                 25s   v1.23.4
```

image のバージョンを上げて以下のように書くといけた。[参考](https://qiita.com/kyontra/items/8e3c0d457d7c1d9e2c69)

```yaml
image: kindest/node:v1.23.0@sha256:49824ab1727c04e56a21a5d8372a402fcd32ea51ac96a2706a12af38934f81ac
```

ただ上記の image は [dockerhub](https://hub.docker.com/layers/node/kindest/node/v1.23.0/images/sha256-2f93d3c7b12a3e93e6c1f34f331415e105979961fcddbe69a4e3ab5a93ccbb35?context=explore) にはなかった

以下のやつを参照すれば良さそう。
https://github.com/kubernetes-sigs/kind/releases

試しに変えてみる

```yaml
image: kindest/node:v1.18.19@sha256:7af1492e19b3192a79f606e43c35fb741e520d195f96399284515f077b3b622c
```

できた

```sh
$ kind create cluster --name kindcluster --config kind.yaml
Creating cluster "kindcluster" ...
 ✓ Ensuring node image (kindest/node:v1.18.19) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kindcluster"
You can now use your cluster with:

kubectl cluster-info --context kind-kindcluster

Thanks for using kind! 😊
```

ひとまず新しそうなやつにしておく。

```yaml
image: kindest/node:v1.21.1@sha256:69860bda5563ac81e3c0057d654b5253219618a22ec3a346306239bba8cfa1a6
```

alpha の方はよくわからんがうまくいかなかった（容量とかのせいか？？）

削除コマンド

```
$ kind delete cluster --name kindcluster
```

ひとまず kind で学習を進めていくことにする。node の数は学習内容に応じて変更する。

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.21.1@sha256:69860bda5563ac81e3c0057d654b5253219618a22ec3a346306239bba8cfa1a6
  - role: worker
    image: kindest/node:v1.21.1@sha256:69860bda5563ac81e3c0057d654b5253219618a22ec3a346306239bba8cfa1a6
  - role: worker
    image: kindest/node:v1.21.1@sha256:69860bda5563ac81e3c0057d654b5253219618a22ec3a346306239bba8cfa1a6
  - role: worker
    image: kindest/node:v1.21.1@sha256:69860bda5563ac81e3c0057d654b5253219618a22ec3a346306239bba8cfa1a6
```

```sh
$ kubectl get no                                       ○ kind-kindcluster
NAME                        STATUS   ROLES                  AGE   VERSION
kindcluster-control-plane   Ready    control-plane,master   88s   v1.21.1
kindcluster-worker          Ready    <none>                 56s   v1.21.1
kindcluster-worker2         Ready    <none>                 57s   v1.21.1
kindcluster-worker3         Ready    <none>                 56s   v1.21.1
```
