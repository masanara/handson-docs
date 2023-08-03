# Kubernetes Handson 1

## 1. kubectlのインストール

踏み台インスタンスにkubectlをダウンロードします。

```bash
cd $HOME; curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
```

ダウンロードしたkubectlを`/usr/local/bin/`に配置します。

```bash
chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl
```

## 2. Kubenetesクラスターに対する接続

kubectlコマンドを実行してクラスターへの接続を確認します。(※本環境では予めkubeconfigを準備してあるためEKSクラスターに接続可能です。)

```bash
kubectl cluster-info
```

クラスターに接続することができれば、クラスターのノードの状態を確認することが可能です。

```bash
kubectl get node -o wide
```

## 3. Namespaceの利用

### 3.1. Namespaceの作成

Kubernetesのハンズオンは、1つのKubernetesクラスターを共有しネームスペースによってリソースを隔離します。各ユーザー用にそれぞれのLinux VMのホスト名を利用してネームスペースを作成します。

ホスト名が`NS`環境変数に設定されていることを確認します。

```bash
echo $NS
```

`NS`の値を名前としてネームスペースを作成します。

```bash
kubectl create namespace $NS
```

### 3.2. Namespaceの確認

ネームスペースが作成されたことを確認します。

```bash
kubectl get namespace
```

### 3.3. コンテクストのNamespaceの変更

現在のcontextを確認します。NAMESPACEとしてdefaultが指定されています。

```bash
kubectl config get-contexts
```

contextのネームスペースを作成したネームスペースに変更します。

```bash
kubectl config set-context --current --namespace=$NS
```

再度contextを確認して、NAMESPACEがdefaultから`NS`の値に変更されていることを確認します。

```bash
kubectl config get-contexts
```

## 4. Pod

### 4.1. Podの作成

作業用ディレクトリを変更します。

```bash
cd $HOME/handson/module1
```

Pod作成用のマニフェストの内容を確認します。

```bash
cat pod.yaml
```

マニフェストの中身は以下のとおりです。Private Registryにある`harbor.nsx.techlab.netone.co.jp/handson/nginx:0.1`をコンテナイメージとして指定し、nginxという名前でPodを作成します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: harbor.nsx.techlab.netone.co.jp/handson/nginx:1.0
    name: nginx
```

マニフェストを利用してPodを作成します。

```bash
kubectl apply -f pod.yaml
```

Podが作成されたことを確認します。`-o wide`オプションをつけることでpodの詳細情報が表示されます。

```bash
kubectl get pods -o wide
```

### 4.2. Pod内のコマンド実行

`kubectl exec`コマンドでPod内でコマンドを実行することが可能です。

Pod内でhostnameコマンドを入力して出力を確認します。Podの名前が出力されます。

```bash
kubectl exec -it nginx -- hostname
```

Pod内でシェル(/bin/bash)を起動することも可能です。

```bash
kubectl exec -it nginx -- /bin/bash
```

Pod内のシェルに切り替わるため、PodのIPアドレスを確認してみます。上記の`kubectl get pods -o wide`で出力された結果とIPアドレスが同じであることが確認できます。

```bash
ip address
```

Pod内の8080番ポートで起動するnginxにcurlでアクセスしてみます。

```
curl localhost:8080
```

`exit`でPodのシェルを終了すると、ターミナルに戻ります。

```bash
exit
```

### 4.3. Podログの確認

`kubectl logs`コマンドでPodのログを確認します。curl でアクセスしたログの記録を確認できます。

```bash
kubectl logs nginx
```

### 4.4. Podの削除

作成したPodを削除します。(Pod内のプロセス停止に数秒かかります)

```bash
kubectl delete pod nginx
```

Podが削除されたことを確認します。

```bash
kubectl get pods
```

## 5. Deployment

### 5.1. Deploymentの作成

Deployment作成用のマニフェストの内容を確認します。

```bash
cat deployment.yaml
```

マニフェストの中身は以下のとおりです。nginxという名前でDeploymentを作成します。DeploymentはDocker Hubにある`netonesystems/nginx:1.0`をコンテナイメージとて、`replicas :3`が指定されているため3つのPodを起動します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: netonesystems/nginx:1.0
        name: nginx
```

マニフェストを利用してDeploymentを作成します。

```bash
kubectl apply -f deployment.yaml
```

Podが作成されたことを確認します。`-o wide`オプションをつけることでpodの詳細情報が表示されます。`nginx-`というプリフィックスがついたPodが3つ作成されていることが確認できます。

```bash
kubectl get pods -o wide
```

### 5.2. ReplicaSetの確認

Deploymentを作成したことにより、ReplicaSetが作成されていることが確認できます。`DESIRED`と`CURRENT`と`READY`が全て3になっていることが確認できます。

```bash
kubectl get replicaset -o wide
```

### 5.3. レプリカ数の変更

`kubectl edit`コマンドで現在のマニフェストをエディタで開き、Deploymentの`.spec.replicas: 3`を`.spec.replicas: 4`に変更しPodが増えることを確認します。(デフォルトではEDITORとしてvimが指定されています。nanoを利用したい場合は`export EDITOR=nano`を実行してから`kubectl edit`を実行してください)

```bash
kubectl edit deploy nginx
```

設定変更を反映したら、Podが増えていることを確認します。

```bash
kubectl get pod -o wide
```

### 5.4. イメージの修正

DeploymentではReplicaSetのバージョンを管理することが可能です。Deploymentで利用するイメージを更新します。

```bash
kubectl set image deploy nginx nginx=harbor.nsx.techlab.netone.co.jp/handson/nginx:2.0
```

上記コマンド実行後に速やかにPodの状態を確認すると、`Terminating`と`Running`のPodが混在しいる様子を確認することが可能です。

```bash
kubectl get pod -o wide
```

ReplicaSetの状態を確認します。新しいReplicaSetが追加され、Podが新しいReplicaSetでREADYとなっていることを確認します。

```bash
kubectl get replicaset
```

### 5.5. Reconcileの確認

以下のコマンドを実行し、Deploymentによって作成されたPodの1つを削除します。(Pod内のプロセス停止に数秒かかります)

```bash
kubectl delete pod $(kubectl get pod -o=jsonpath='{.items[1]..metadata.name}' -l app=nginx)
```

Podの状態を確認します。1つのPodが削除されますが、ReplicaSetにより新たなPodが起動するため、1つのPodだけAGEが若いことが確認できます。

```bash
kubectl get pod -o wide
```

### 5.6. レプリカ数の変更

`kubectl scale`コマンドでDeploymentのReplica数を修正します。5.3. レプリカ数の変更では マニフェストを編集しましたが、CLIから数を修正することも可能です。

```bash
kubectl scale deploy nginx --replicas=2
```

コマンド実行後、nginx Podが2に減少することを確認できます。

```bash
kubectl get pod -o wide
```

---

[戻る](handson.html)
