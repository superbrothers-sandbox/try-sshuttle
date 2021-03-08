VM を起動します。

```sh
$ vagrant up
```

ここでは bastion サーバ (192.168.33.10) と k8s サーバ (192.168.33.11) が作成されます。今回は bastion サーバ経由で k8s サーバ上に構築されたクラスタにアクセスするというシナリオです。

```
┌────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│                │        │                 │        │                 │
│   localhost    ├───────►│    bastion VM   ├───────►│     k8s VM      │
│                │        │                 │        │                 │
└────────────────┘        └─────────────────┘        └─────────────────┘
                             192.168.33.10              192.168.33.11
```

sshuttle を実行します。sshuttle をインストールしていない場合は https://github.com/sshuttle/sshuttle#obtaining-sshuttle を参考にインストールしてください。

```sh
$ sshuttle --ssh-cmd "ssh -o UserKnownHostsFile=/dev/null -i $(pwd)/.vagrant/machines/bastion/virtualbox/private_key" -r vagrant@127.0.0.1:2200 192.168.33.11/32
```

- `--ssh-cmd` のオプションは、bastion サーバに素直に ssh できる環境であれば不要です。ここで設定しているのは Vagrant で作成した VM ならではの設定です。
- `-r` でリモートのサーバ、つまりここでは bastion サーバ経由でアクセスするためにユーザ名に `vagrant` でアドレスに `127.0.0.1`、ポートに `2200` を指定しています。Vagrant で作成した VM の SSH ポートは人によって異なる可能性があるため、接続できない場合は `vagrant ssh-config bastion` コマンドでポート等を確認してください。
- 最後の `192.168.33.11/32` がトラフィックを転送するサブネットです。ここでは Kubernetes クラスタに接続したいだけなので `k8s` VM のアドレスを指定しています。パブリッククラウドの場合は Kubernetes クラスタのエンドポイントのアドレスを指定してください。

この設定でローカルからの `192.168.33.11` のトラフィックが bastion 経由で送信されます。仕組みは簡単で、SSH ポート転送を使いつつ、iptables（macOS の場合は ipfw）で `192.168.33.11` のトラフィックを `127.0.0.1` に転送しています。

次からの作業は別ターミナルから実施してください。

---

k8s サーバからアドミンの kubeconfig ファイルを取得します。

```sh
# Download admin kubeconfig file from the k8s vm
$ vagrant ssh k8s -c "sudo cat /etc/kubernetes/admin.conf" >./admin.conf
```
この設定ファイルでは `192.168.33.11:6443` で Kubernetes クラスタに接続するようになっています。sshuttle を使用していない場合はもちろん直接接続はできません。パブリッククラウドの場合は、通常のフローでローカルで認証情報を取得してください。

クラスタに接続してみます。問題なく接続できたら成功です。

```sh
$ export KUBECONFIG=$(pwd)/admin.conf
$ kubectl get node
NAME           STATUS   ROLES                  AGE     VERSION
ubuntu-focal   Ready    control-plane,master   3m16s   v1.20.4
```
