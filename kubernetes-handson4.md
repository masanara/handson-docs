# Kubernetes Handson 4

## 13. StorageClassの確認

本環境はAWSのEC2インスタンスであるため、永続ボリュームはAmazon EBSによって実現されます。StorageClassを確認するとgp2が作成されており、デフォルトとして設定されています。

```bash
$ kubectl get storageclass
NAME                    PROVISIONER              RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
unity-k8s (default)     csi.vsphere.vmware.com   Delete          Immediate              true                   8d
unity-k8s-latebinding   csi.vsphere.vmware.com   Delete          WaitForFirstConsumer   true                   8d
```

## 14. PersistentVolumeClaimの作成

作業用ディレクトリを変更します。

```bash
cd $HOME/handson/module4
```

PVC作成用のマニフェストの内容を確認します。

```bash
cat pvc.yaml
```

StorageClass `unity-k8s`に対してアクセスモード ReadWriteOnce、1Giの容量のボリュームを要求します。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: unity-k8s
```

マニフェストを利用してPVCを作成します。

```bash
kubectl apply -f pvc.yaml
```

作成したPVCを確認すると、`Pending`状態となっています。これはStorageClass : gp2のVolumeBindingModeがWaitForFirstConsumerとして構成されているため、PVCを利用するPodが作成されるタイミングでPVCがプロビジョニングされるためです。

```bash
kubectl get pvc
```

## 15. PVCの利用

### 15.1. PVCを利用するPodの作成

Podを作成するためのマニフェストを確認します。

```bash
cat pod.yaml
```

Podは作成済みのmypvcという名前のPVCリソースをVolumeとして利用し、`/tmp/data`にマウントして起動します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: busybox
      image: harbor.nsx.techlab.netone.co.jp/handson/busybox:latest
      volumeMounts:
      - name: pvc
        mountPath: "/tmp/data"
      command: ["sleep", "3600"]
  volumes:
    - name: pvc
      persistentVolumeClaim:
        claimName: mypvc
```

マニフェストを利用してPodを作成します。

```bash
kubectl apply -f pod.yaml
```

作成したPodの状態を確認します。暫く待つと、Pod mypodがRunning状態になります。

```bash
kubectl get pod
```

### 15.2. PVCに対するデータの書き込み

起動したPodにexecして、`/tmp/data`にファイルを作成します。

```bash
kubectl exec -it mypod -- sh -c 'date > /tmp/data/txt1'
```

ファイルが作成されており、読み込み可能なことを確認します。

```bash
kubectl exec -it mypod -- ls -l /tmp/data/txt1
```

```bash
kubectl exec -it mypod -- cat /tmp/data/txt1
```

### 15.3. Podの削除とボリューム永続性の確認

Podを削除して、ボリュームがPodから独立して管理できることを確認します。

```bash
kubectl delete -f pod.yaml
```

Pod削除されたことを確認します。

```bash
kubectl get pod
```

PVCが残存していることを確認します。

```bash
kubectl get pvc
```

再度Podを作成します。

```bash
kubectl apply -f pod.yaml
```

Podが起動したことを確認します。

```bash
kubectl get pod
```

起動したPodにexecして、`/tmp/data`のファイルが存在していることを確認します。

```bash
kubectl exec -it mypod -- ls -l /tmp/data/txt1
```

```
kubectl exec -it mypod -- cat /tmp/data/txt1
```

## 16. リソースの削除

PodとPVCを削除します。(削除が完了するまで数秒かかります)

```bash
kubectl delete pod/mypod pvc/mypvc
```

PodとPVCが削除されたことを確認します。

```bash
kubectl get pod,pvc
```

## 17. StatefulSet

### 17.1. StatefulSetの作成

StatefulSetを作成するためのマニフェストを確認します。

```bash
cat statefulset.yaml
```

StatefulSetは`k8s.gcr.io/nginx-slim:0.8`をコンテナイメージとして利用し、StorageClass `gp2`に対してアクセスモード ReadWriteOnce、1Giの容量のボリュームを要求します。作成されたボリュームは、nginx Podの`/usr/share/nginx/html`にマウントされます。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: unity-k8s
      resources:
        requests:
          storage: 1Gi
```

マニフェストを利用してPodを作成します。

```bash
kubectl apply -f statefulset.yaml
```

作成したStatefulSet, Pod, PVCの状態を確認します。暫く待つと、Stetefulset webがREADYとなり、Pod web-0がRunning状態となり、PVCとしてwww-web-0が作成されます。

```bash
kubectl get statefulset,pod,pvc
```

出力内容の例 : StatefulSetの作成によりPodとPVCが作成されています。

```bash
$ kubectl get statefulset,pod,pvc
NAME                   READY   AGE
statefulset.apps/web   1/1     57s

NAME        READY   STATUS    RESTARTS   AGE
pod/web-0   1/1     Running   0          57s

NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/www-web-0   Bound    pvc-0774e315-e03f-4fe8-b7e7-0c91644e387e   1Gi        RWO            unity-k8s      57s
```

### 17.2. (Option) PVCによりデータ永続性の確認

作成されたweb-0 Podのシェルを利用してボリュームにデータを書き込んだ後に、Podを削除してみましょう。

作成済みのPVCをマウントしたPodが自動で再作成され、PVCのデータが保持されていることを確認してみましょう。

## 18. リソースの削除

StatefulSetを削除します。

```bash
kubectl delete -f statefulset.yaml
```

StatefulSetを削除しても、PVCは維持されるためPVCを削除します。

```bash
kubectl delete pvc www-web-0
```

---

[戻る](handson.html)
