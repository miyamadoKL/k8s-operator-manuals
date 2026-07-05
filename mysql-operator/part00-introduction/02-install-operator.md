# 第2章 Operator のインストール

> 本章で参照する公式リソース
>
> - [deploy/deploy-crds.yaml L1-L10](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/deploy/deploy-crds.yaml#L1-L10)
> - [deploy/deploy-operator.yaml L130-L163](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/deploy/deploy-operator.yaml#L130-L163)
> - [helm/mysql-operator/Chart.yaml L1-L6](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/Chart.yaml#L1-L6)
> - [helm/mysql-operator/values.yaml L1-L21](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/values.yaml#L1-L21)

## この章でできるようになること

kubectl と Helm の2通りの方法で MySQL Operator をインストールし、Operator が起動していることを確認できるようになる。

## 前提

Kubernetes 1.21 以上のクラスタと、`kubectl` が使える環境を用意する。
Helm でインストールする場合は Helm v3 も必要である。

## インストール方法の選び方

MySQL Operator のインストール方法には、`kubectl apply` による方法と Helm chart による方法の2通りがある。
どちらも最終的に同じ CRD と Operator Deployment を作成するが、Helm のほうがイメージリポジトリやプルポリシーの上書きを values で管理しやすい。

## 方法1 kubectl による直接インストール

CRD 定義と Operator 本体は別々の manifest ファイルに分かれている。

[deploy/deploy-crds.yaml L1-L10](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/deploy/deploy-crds.yaml#L1-L10)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: innodbclusters.mysql.oracle.com
spec:
  group: mysql.oracle.com
  versions:
    - name: v2
      served: true
      storage: true
```

先に CRD を登録してから、Operator の Deployment を作成する。

```console
$ kubectl apply -f https://raw.githubusercontent.com/mysql/mysql-operator/8.4.9-2.1.11/deploy/deploy-crds.yaml
customresourcedefinition.apiextensions.k8s.io/innodbclusters.mysql.oracle.com created
customresourcedefinition.apiextensions.k8s.io/mysqlbackups.mysql.oracle.com created
customresourcedefinition.apiextensions.k8s.io/clusterkopfpeerings.zalando.org created
customresourcedefinition.apiextensions.k8s.io/kopfpeerings.zalando.org created

$ kubectl apply -f https://raw.githubusercontent.com/mysql/mysql-operator/8.4.9-2.1.11/deploy/deploy-operator.yaml
clusterrole.rbac.authorization.k8s.io/mysql-operator created
clusterrole.rbac.authorization.k8s.io/mysql-sidecar created
clusterrolebinding.rbac.authorization.k8s.io/mysql-operator-rolebinding created
clusterkopfpeering.zalando.org/mysql-operator created
namespace/mysql-operator created
serviceaccount/mysql-operator-sa created
deployment.apps/mysql-operator created
```

`deploy-operator.yaml` は、Operator 専用の namespace `mysql-operator` の作成、ClusterRole、ClusterRoleBinding、ServiceAccount の作成、そして Operator 本体の Deployment 作成までを含む。
Deployment 部分を確認すると、Operator コンテナのイメージとバージョンラベルが本書の対象タグと一致していることがわかる。

[deploy/deploy-operator.yaml L130-L163](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/deploy/deploy-operator.yaml#L130-L163)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-operator
  namespace: mysql-operator
  labels:
    version: "1.0"
    app.kubernetes.io/name: mysql-operator
    app.kubernetes.io/instance: mysql-operator
    app.kubernetes.io/version: 8.4.9-2.1.11
    app.kubernetes.io/component: controller
    app.kubernetes.io/managed-by: mysql-operator
    app.kubernetes.io/created-by: mysql-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      name: mysql-operator
  template:
    metadata:
      labels:
        name: mysql-operator
    spec:
      containers:
        - name: mysql-operator
          image: container-registry.oracle.com/mysql/community-operator:8.4.9-2.1.11
          imagePullPolicy: IfNotPresent
          args:
            [
              "mysqlsh",
              "--log-level=@INFO",
              "--pym",
              "mysqloperator",
              "operator",
            ]
```

Operator コンテナは `mysqlsh`（MySQL Shell）を起点に `mysqloperator` パッケージの `operator` サブコマンドとして起動する。
`replicas: 1` なので、Operator 自体は単一の Pod として動く。
複数の InnoDBCluster を1つの Operator が並行して監視する構成になる。

## 方法2 Helm chart によるインストール

Helm chart の Chart.yaml は、chart バージョンと appVersion（対象タグ）を示す。

[helm/mysql-operator/Chart.yaml L1-L6](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/Chart.yaml#L1-L6)

```yaml
apiVersion: v2
appVersion: "8.4.9-2.1.11"
name: mysql-operator
description: MySQL Operator Helm Chart for deploying MySQL InnoDB Cluster in Kubernetes
type: application
version: "2.1.11"
```

values.yaml でカスタマイズできる主なフィールドは、イメージのレジストリ、リポジトリ、プルポリシーである。

[helm/mysql-operator/values.yaml L1-L21](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/values.yaml#L1-L21)

```yaml
image:
  registry: container-registry.oracle.com
  repository: mysql
  name: community-operator
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: ""
  pullSecrets:
    enabled: false
    secretName:

envs:
    imagesPullPolicy: IfNotPresent
    imagesDefaultRegistry: 
    imagesDefaultRepository:
    k8sClusterDomain:

# If you would like to debug the Helm output with `helm template`, you need
# to turn disableLookups on as during `helm template` Helm won't contact the kube API
# and all lookups will thus fail
disableLookups: false
```

`image.tag` を空にした場合、chart の `appVersion`（`8.4.9-2.1.11`）がそのままイメージタグとして使われる。
`envs.imagesDefaultRegistry` と `envs.imagesDefaultRepository` は、InnoDBCluster が生成する MySQL Server や Router のイメージ既定値を上書きするための環境変数であり、Operator 自体のイメージ指定である `image` セクションとは役割が異なる。

Helm リポジトリを登録してからインストールする。

```console
$ helm repo add mysql-operator https://mysql.github.io/mysql-operator/
"mysql-operator" has been added to your repositories

$ helm repo update

$ helm install mysql-operator mysql-operator/mysql-operator \
    --namespace mysql-operator --create-namespace
NAME: mysql-operator
NAMESPACE: mysql-operator
STATUS: deployed
```

## インストール後の確認

CRD が登録されたことを確認する。

```console
$ kubectl get crd innodbclusters.mysql.oracle.com mysqlbackups.mysql.oracle.com
NAME                              CREATED AT
innodbclusters.mysql.oracle.com   2026-07-01T00:00:00Z
mysqlbackups.mysql.oracle.com     2026-07-01T00:00:00Z
```

Operator の Deployment が Ready になったことを確認する。

```console
$ kubectl -n mysql-operator get deployment mysql-operator
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
mysql-operator   1/1     1            1           30s
```

`READY` が `1/1` になれば、Operator Pod の readinessProbe が通過し、InnoDBCluster リソースの監視を開始した状態である。

## トラブルシューティング

Operator Pod が `ImagePullBackOff` になる場合は、`kubectl -n mysql-operator describe pod` でイメージ名を確認し、`container-registry.oracle.com` への到達性を確認する。
社内プロキシ経由でレジストリにアクセスする環境では、Helm の `image.registry` を社内ミラーに書き換えて再インストールする。

## まとめ

kubectl による直接適用と Helm chart のどちらでも、最終的に同じ CRD 一式と `mysql-operator` namespace 内の Operator Deployment が作成される。
Helm を使うと、イメージのレジストリやプルポリシーを values.yaml 経由で上書きしやすい。

## 関連する章

- [第1章 MySQL Operator の概要とアーキテクチャ](01-overview.md)
- [第3章 クイックスタート](03-quickstart.md)
