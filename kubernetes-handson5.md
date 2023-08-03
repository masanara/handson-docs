# Kubernetes Handson 5

## 19. Wordpressの構築

CMS(Content Management System)であるWordpressをKubernetes環境で構築します。Wodpressはphpで実装されており、バックエンドデータベースとしてMySQLを利用します。本ハンズオンではwordpressとMySQLをそれぞれPodとして作成し、Serviceによりクラスター外に公開します。

以下の要件を満たすようマニフェストを作成し、実装してみましょう。Wodrepss及びMySQLが利用する各種環境変数はSecret/ConfigMapを利用して実装してみましょう。

>  ハンズオンフォルダ($HOME/handson/module5)に回答例があります。

![image-20211128204032586](kubernetes-handson5.assets/image-20211128204032586.png)

### wodpress

- Deployment
  - コンテナイメージ : harbor.nsx.techlab.netone.co.jp/handson/wordpress:6.2.2-php8.0-apache ([wordpress:6.2.2-php8.0-apache](https://hub.docker.com/layers/library/wordpress/6.2.2-php8.2-apache/images/sha256-47bcdee69d620e5d33795ff9b6154ba3a292440f6769a8ff2dbf645625c475fe?context=explore))
  - レプリカ数 : 1
  - 環境変数
    - データベースホスト名 : WORDPRESS_DB_HOST
    - データベース名 : WORDPRESS_DB_NAME
    - データベースユーザー名 : WORDPRESS_DB_USER
    - データベースパスワード : WORDPRESS_DB_PASSWORD
- Service
  - ポート : 80
  - Type : LoadBalancer

### Mysql

- StatefulSet
  - コンテナイメージ :  harbor.nsx.techlab.netone.co.jp/handson/mysql:5.7.43 ([mysql:5.7.43](https://hub.docker.com/layers/library/mysql/5.7.43/images/sha256-aaa1374f1e6c24d73e9dfa8f2cdae81c8054e6d1d80c32da883a9050258b6e83?context=explore))
  - レプリカ数 : 1
  - 環境変数
    - 管理者パスワードの設定 : MYSQL_RANDOM_ROOT_PASSWORD
    - データベース名 : MYSQL_DATABASE
    - データベースユーザー名 : MYSQL_USER
    - データベースパスワード : MYSQL_PASSWORD
  - ボリューム
    - マウントパス : /var/lib/mysql
    
    - EKS の永続ボリュームではファイルシステムのルートに lost+found のディレクトリが存在するため、 MySQL コンテナの初期化処理がエラーとなってしまい、異常終了します。そのため、subPath を設定し、空のディレクトリをマウントするようにします。
    
      ```yaml
          volumeMounts:
          - mountPath: /var/lib/mysql
            name: mysql-data
            subPath: mysql 
      ```
  - ボリューム要求
    - アクセスモード : ReadWriteOnce
    - StorageClass : unity-k8s
    - ストレージ容量 : 2Gi
  
- Service
  - ポート : 3306
  - Type : ClusterIP

---

[戻る](handson.html)