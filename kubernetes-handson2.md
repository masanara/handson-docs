# Kubernetes Handson 2

## 6 環境変数の利用

### 6.1. 環境変数を利用したPodの起動

作業用ディレクトリを変更します。

```bash
cd $HOME/handson/module2
```

Pod作成用のマニフェストの内容を確認します。

```bash
cat env-test1.yaml
```

Docker Hubにある`busybox`をコンテナイメージとして指定し、環境変数`MESSAGE`の内容を標準出力に表示し、3秒間sleepを繰り返します。

```bash
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: env-test1
  name: env-test1
spec:
  containers:
  - image: harbor.nsx.techlab.netone.co.jp/handson/busybox:latest
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(MESSAGE); sleep 3; done"]
    name: busybox
    env:
    - name: MESSAGE
      value: "Hello World"
```

マニフェストを利用してPodを起動します。

```bash
kubectl apply -f env-test1.yaml
```

### 6.2. 起動したPodの確認

pod(env-test1)が起動したことを確認します。

```bash
kubectl get pod
```

`.spec.containers[].env`で、環境変数`MESSAGE`に「Hello World」という文字列を設定しているため、Podのログとして3秒毎に「Hello World」が表示されます。

```bash
kubectl logs -f env-test1
```

`kubectl logs`コマンドに`-f`オプションをつけることで、追加されるログを継続的に観測することが可能です。3秒毎にログが出力されることを確認したら、Ctrl+cを入力してログ出力を停止します。

## 7. ConfigMapの利用

### 7.1. ConfigMapの作成

ConfigMap作成用のマニフェストの内容を確認します。

```bash
cat configmap.yaml
```

マニフェストの中身は以下のとおりです。`ENV`というキーに対して「Value from ConfigMap」という値を設定しています。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-env
data:
  ENV: Value from ConfigMap
```

マニフェストを利用してConfigMapを作成します。

```bash
kubectl apply -f configmap.yaml
```

ConfigMapが作成されたことを確認します。

```bash
kubectl get configmap
```

ConfigMapの中身を確認します。マニフェストと同じdataが含まれていることが確認できます。

```bash
kubectl get configmap cm-env -oyaml
```

### 7.2 ConfigMapを利用するPodの起動

作成したConfigMap (cm-env)を参照するPodを起動するためのマニフェストの内容を確認します。

```bash
cat env-test2.yaml
```

Docker Hubにある`busybox`をコンテナイメージとして指定し、環境変数`MESSAGE`の内容を標準出力に表示し、3秒間sleepを繰り返します。`.spec.containers[].env`として`MESSAGE`環境変数を設定しています。環境変数の値はConfigMap (cm-env)を参照し、cm-envの`ENV`キーに対する値(Value from ConfigMap)を利用します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: env-test2
  name: env-test2
spec:
  containers:
  - image: harbor.nsx.techlab.netone.co.jp/handson/busybox:latest
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(MESSAGE); sleep 3; done"]
    name: busybox
    env:
    - name: MESSAGE
      valueFrom:
        configMapKeyRef:
          name: cm-env
          key: ENV
```

マニフェストを利用してPodを起動します。

```bash
kubectl apply -f env-test2.yaml
```

pod(env-test2)が起動したことを確認します。

```bash
kubectl get pod
```

`.spec.containers[].env`で、環境変数`MESSAGE`にConfigMap(cm-env)で指定している`ENV`の値として「Value from ConfigMap」という文字列を設定しているため、Podのログとして3秒毎に「Value from ConfigMap」が表示されます。

```bash
kubectl logs -f env-test2
```

3秒毎にログが出力されることを確認したら、Ctrl+cを入力してログ出力を停止します。

## 8. Secretの利用

### 8.1. Secretの作成

ConfigMap作成用のマニフェストの内容を確認します。

```bash
cat secret.yaml
```

マニフェストの中身は以下のとおりです。`USERNAME`と`PASSWORD`というキーに対してBase64でエンコードされた文字列が設定されています。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mypassword
type: Opaque
data:
  PASSWORD: cGFzc3dvcmQ=
  USERNAME: dXNlcjE=
```

マニフェストを利用してSecretを作成します。

```bash
kubectl apply -f secret.yaml
```

Secretが作成されたことを確認します。

```bash
kubectl get secret
```

Secretの中身を確認します。マニフェストと同じdataが含まれていることが確認できます。

```bash
kubectl get secret mypassword -oyaml
```

データに含まれている文字列を取り出し、Base64でデコードすると、文字列の内容を確認することが可能です。

> kubectl getコマンド実行時に `-o=jsonpath`を指定すると[JSONPath](https://kubernetes.io/ja/docs/reference/kubectl/jsonpath/)の指定により、特定の値だけを出力することが可能です。以下のコマンドはそれぞれUSERNAMEとPASSWORDの値だけを取り出して、base64でデコードしています。

```bash
kubectl get secret mypassword -o=jsonpath='{.data.USERNAME}' | base64 -d
```

```bash
kubectl get secret mypassword -o=jsonpath='{.data.PASSWORD}' | base64 -d
```

### 8.2 Secretを利用するPodの起動

作成したSecret (mypassword)を参照するPodを起動するためのマニフェストの内容を確認します。

```bash
cat env-test3.yaml
```

Docker Hubにある`busybox`をコンテナイメージとして指定し、環境変数`USERNAME`と`PASSWORD`の内容を標準出力に表示し、3秒間sleepを繰り返します。`.spec.containers[].envFrom`としてSecret (mypassword)で指定されている値を環境変数として設定しています。環境変数はmypasswordの`USERNAME`と`PASSWORD`を参照しています。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: env-test3
  name: env-test3
spec:
  containers:
  - image: harbor.nsx.techlab.netone.co.jp/handson/busybox:latest
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo username=$(USERNAME) / password=$(PASSWORD); sleep 3; done"]
    name: busybox
    envFrom:
    - secretRef:
        name: mypassword
```

マニフェストを利用してPodを起動します。

```bash
kubectl apply -f env-test3.yaml
```

Pod(env-test3)が起動したことを確認します。

```bash
kubectl get pod
```

環境変数`USERNAME`と`PASSWORD`Secret(mypassword)で指定している「user1」と「password」を設定しているため、Podのログとして3秒毎に「username=user1 / password=password」が表示されます。

```bash
kubectl logs -f env-test3
```

3秒毎にログが出力されることを確認したら、Ctrl+cを入力してログ出力を停止します。

## 9. リソースの削除

Handson 2で作成したマニフェストが含まれるディレクトリを指定して`kubectl delete`を実行することでHandson2で作成した全てのリソースを削除します。(※Handson1で作成したリソース、後ほど利用するため削除しません)

作業用ディレクトリを変更します。

```bash
cd $HOME/handson/module2
```

カレントディレクトリにあるマニフェストファイルにあるリソースを全て削除します。

```bash
kubectl delete -f .
```

Handson 2で作成したConfigMap/Secret/Podが削除されたことを確認します。(複数のリソース種類ををカンマで区切りで続けて指定すると、一度に確認することが可能です。)

```bash
kubectl get configmap,secret,pod
```

---

[戻る](handson.html)
