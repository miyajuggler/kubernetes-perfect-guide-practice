## 学習を始める前の環境

```sh
$ kubectl get node
NAME             STATUS   ROLES                  AGE    VERSION
docker-desktop   Ready    control-plane,master   117d   v1.21.2

$ kubectl cluster-info
Kubernetes control plane is running at https://kubernetes.docker.internal:6443
CoreDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

$ kubectl version
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.3", GitCommit:"ca643a4d1f7bfe34773c74f79527be4afd95bf39", GitTreeState:"clean", BuildDate:"2021-07-15T21:04:39Z", GoVersion:"go1.16.6", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.2", GitCommit:"092fbfbf53427de67cac1e9fa54aaa09a28371d7", GitTreeState:"clean", BuildDate:"2021-06-16T12:53:14Z", GoVersion:"go1.16.5", Compiler:"gc", Platform:"linux/arm64"}

$ kubectl version --short --client
Client Version: v1.21.3

$ kubectl config current-context
docker-desktop
```

### kubernetes を利用する際も docker について必ず学んておいたほうが良い知識

- コンテナの設計
- dockerfile の書き方
- イメージのビルド
- レジストリへのイメージプッシュ

### コンテナの設計で大事なこと

- 1 コンテナにつき 1 プロセス
- immutable infrastructure なイメージにする
  - 不変のインフラ（）
- 軽量な docker イメージにする
- 実行ユーザーを root 以外にする

### マルチステージビルドの dockerfile

普通に書くと以下になる。

```dockerfile
# Alpine 3.11ベースのgolang 1.14.1のイメージをベースとして使用
FROM golang:1.14.1-alpine3.11

# ビルドを行うマシン上のmain.goファイルをコンテナにコピー
COPY ./main.go ./

# ビルド時にコンテナ内でコマンドを実行
RUN go build -o ./go-app ./main.go

# 実行ユーザを指定
USER nobody

# コンテナ起動時に実行するコマンドを定義
ENTRYPOINT ["./go-app"]
```

docker build (docker image build)

```sh
$ docker image ls
REPOSITORY        TAG   IMAGE ID       CREATED          SIZE
sample-image      0.1   6101110fdc81   30 seconds ago   371MB
```

しかしこれだとサイズが無駄にでかい。中身としては golang という Go のコンパイルツールが有るためこれだけでかくなる。  
しかし、これはアプリケーションのコンパイル時に必要とされるだけで、成果物の実行時には必要がない。  
そのため以下のように書き直す。

```dockerfile
# Stage 1のコンテナ（アプリケーションをビルド）
FROM golang:1.14.1-alpine3.11 as builder
COPY ./main.go ./
RUN go build -o /go-app ./main.go

# Stage 2のコンテナ（ビルドしたバイナリを内包した実行用コンテナを作成）
FROM alpine:3.11
# Stage 1でビルドした成果物をコピー
COPY --from=builder /go-app .
ENTRYPOINT ["./go-app"]
```

1 段階目で golang を用いて Go 言語で書かれたアプリケーションをコンパイルする。  
2 段階目で 1 段階目でコンパイルした成果物を内包して実行用のコンテナを作成する。

これで

- ソースコードのコンパイルに必要なツールを、実際にアプリケーションを起動させるコンテナに内包しないため軽い。
- 不要なソフトウェアをインストールしないため、セキュリティの観点からもメリットがある。

```sh
# マルチステージ用の dockerfile を指定する
$ docker image build -t sample-image:0.2 -f Dockerfile-MultiStage .

$ docker image ls
REPOSITORY        TAG   IMAGE ID       CREATED          SIZE
sample-image      0.2   6f65e66ef46e   7 seconds ago    12.5MB
sample-image      0.1   6101110fdc81   29 minutes ago   371MB
```

### 他

また、レイヤごとのファイルの書き換えが多い場合、 `docker image build` 時に `--squash` をつけることで、最終的なファイル状態を持った 1 つのレイヤにまとめられる。故に image の縮小につながる。

### image をもとにコンテナを立てる

```sh
# -d で detached mode(バックグラウンドで動かす)、-p でポート指定
$ docker image run -d -p 8888:8080 sample-image:0.1

$ curl http://localhost:8888
Hello, Kubernetes%
```
