# live-rec-k8s
ライブ配信の自動録画サーバー環境をKubernetes（[Apache Kafka](https://kafka.apache.org/), [Argo Workflows](https://argoproj.github.io/workflows/), [MinIO](https://min.io/)）で構築する。

## 手順

### 0. 事前準備

#### 0.1 git clone

```sh
git clone git@github.com:genkaieng/live-rec-k8s.git
```

#### 0.2 Kubernetes環境を用意する

なんでも良いけど [minikube](https://minikube.sigs.k8s.io/) を使う場合

```sh
minikube start
```

ダッシュボードを表示

```sh
minikube dashboard
```

### 1. Kubernetesにデプロイする

デプロイには [helm](https://helm.sh/) を利用する。

Apache KafkaとArgo WorkflowsとMinIOに依存するのでチャートを落としてくる。

```sh
helm dependencies update
```

デプロイする。

```sh
helm install my-release .
```

正常にデプロイが行われたら、Apache KafkaとArgo WorkflowsとMinIOの環境が出来上がってることがダッシュボードで確認できるはず。

ポートフォワーディングでArgo WorkflowsとMinIOにブラウザでアクセス出来る。

#### Argo Workflows
```sh
kubectl port-forward svc/my-release-argo-workflows-server 2746:2746
```

#### MinIO

```sh
kubectl port-forward svc/my-release-minio 9001:9001
```

MinIOのログイン情報を確認する

```sh
# ユーザー名を表示
kubectl get secret my-release-minio -o jsonpath="{.data.root-user}" | base64 --decode

# パスワードを表示
kubectl get secret my-release-minio -o jsonpath="{.data.root-password}" | base64 --decode
```

### 2. MinIOのバケットとアクセスキーを作成

Argo WorkflowsのArtifact Repositoryに利用するバケットを作成、及びCredentialsの設定を行っていきます。

#### 2.1 バケットを作成

ブラウザでMinIO管理画面にログインできたら、左ペインからBucketsを開き、右上のCreate Bucketボタンを押下。

バケット作成画面を表示できたら、バケット名を **mybucket** としてCreate Bucketボタンを押下。

<!--![Screenshot from 2024-02-24 01-22-51](https://github.com/genkaieng/live-rec-k8s/assets/154831542/3d28a750-0f44-4d91-910e-37901bbc046e) -->

バケットが作成される。

![Screenshot from 2024-02-24 01-25-47](https://github.com/genkaieng/live-rec-k8s/assets/154831542/6d1094eb-cdcc-40e5-8d30-cb7990e12a21)

#### 2.2 アクセスキーの作成

MinIO管理画面の左ペインからAccess Keysを開いて、Create access keyボタンを押下。

アクセスキー作成画面が開くので、下のCreateボタンを押下。

**Access Key**と**Secret Key**がポップで表示される。

![Screenshot from 2024-02-24 01-35-14](https://github.com/genkaieng/live-rec-k8s/assets/154831542/f4afe247-9a7e-4225-b166-479103cd59fd)

これをKubernetesにsecretで保存する。

```
kubectl create secret generic minio-credentials \
--from-literal=accesskey=<Access Key> \
--from-literal=secretkey=<Secret Key>
```

#### 2.3 アクセスキーにポリシーを設定

作成したアクセスキーをクリックすると設定変更のポップが開くので、以下のポリシーを設定して保存する。

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::*"
            ]
        }
    ]
}
```
