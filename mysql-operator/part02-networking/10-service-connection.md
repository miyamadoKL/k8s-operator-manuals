# 第10章 Service と接続エンドポイント

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L324-L345](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L324-L345)（`spec.service`）
> - [helm/mysql-operator/crds/crd.yaml#L860-L863](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L860-L863)（`spec.serviceFqdnTemplate`）
> - [mysqloperator/controller/innodbcluster/cluster_objects.py#L34-L63](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L34-L63)（headless Service の生成）

## この章でできるようになること

InnoDB Cluster が作成する複数の Service を区別し、用途に応じて正しい接続先を選べるようになる。
`spec.service.defaultPort` の意味を理解し、アプリケーションからの接続を書き込み用途と読み取り用途に振り分けられるようになる。

## 前提

- [InnoDBCluster リソースの全体像](../part01-innodbcluster-basics/04-innodbcluster-resource.md)で Custom Resource を作成済みであること。

## Operator が作成する Service

Operator は InnoDBCluster ごとに、役割の異なる2種類の Service を作成する。

1つ目は `<cluster名>-instances` という名前の headless Service である。
これは MySQL インスタンスの Pod 1台ずつに直接到達するための Service で、`clusterIP: None` かつ `publishNotReadyAddresses: true` として作成される。

[cluster_objects.py の headless Service テンプレート](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L34-L63)は次のとおりである。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {spec.name}-instances
  namespace: {spec.namespace}
  labels:
    tier: mysql
    mysql.oracle.com/cluster: {spec.cluster_name}
    mysql.oracle.com/instance-type: {instance_type}
spec:
  clusterIP: None
  publishNotReadyAddresses: true
  ports:
  - name: mysql
    port: {spec.mysql_port}
    targetPort: {spec.mysql_port}
  - name: mysqlx
    port: {spec.mysql_xport}
    targetPort: {spec.mysql_xport}
  - name: gr-xcom
    port: {spec.mysql_grport}
    targetPort: {spec.mysql_grport}
  selector:
    component: mysqld
    tier: mysql
    mysql.oracle.com/cluster: {spec.cluster_name}
    mysql.oracle.com/instance-type: {instance_type}
  type: ClusterIP
```

`publishNotReadyAddresses: true` になっている理由は、Group Replication の初期化中や再構成中の Pod にも到達できる必要があるためである。
readiness を条件にすると、クラスタ構築の途中でメンバー間通信が届かず、リコンサイルが進まなくなる。
このため、アプリケーションからの通常の接続にこの Service を使うのは避け、後述の Router 経由の接続を使う。

2つ目は `<cluster名>` という、Custom Resource と同名の Service である。
これは [第11章 MySQL Router](11-mysql-router.md)で扱う MySQL Router の Pod へ到達するための Service であり、アプリケーションが通常利用するのはこちらになる。

```bash
kubectl get service -l mysql.oracle.com/cluster=mycluster
```

```text
NAME                 TYPE        CLUSTER-IP     PORT(S)
mycluster            ClusterIP   10.96.10.20    3306/TCP,33060/TCP,...
mycluster-instances  ClusterIP   None           3306/TCP,33060/TCP,33061/TCP
```

`mycluster` が Router 向け、`mycluster-instances` が headless の直接到達用である。

## Service のカスタマイズ（spec.service）

`spec.service` は、`mycluster` という Router 向け Service の `type` や付加情報を制御する。

[crd.yaml の spec.service](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L324-L345)は次のとおりである。

```yaml
service:
  type: object
  description: "Configuration of the Service used by applications connecting to the InnoDB Cluster"
  properties:
    type:
      type: string
      enum: ["ClusterIP", "NodePort", "LoadBalancer"]
      default: ClusterIP
    annotations:
      type: object
      description: "Custom annotations for the Service"
      x-kubernetes-preserve-unknown-fields: true
    labels:
      type: object
      description: "Custom labels for the Service"
      x-kubernetes-preserve-unknown-fields: true
    defaultPort:
      type: string
      description: "Target for the Service's default (3306) port. If mysql-rw traffic will go to the primary and allow read and write operations, with mysql-ro traffic goes to the replica and allows only read operations, with mysql-rw-split the router's read-write-splitting will be targeted"
      enum: ["mysql-rw", "mysql-ro", "mysql-rw-split"]
      default: "mysql-rw"
```

| フィールド | 説明 |
|---|---|
| `type` | Service の種別。クラウド環境で外部公開する場合は `LoadBalancer` を指定する |
| `annotations` | Service に付与する任意の annotation |
| `labels` | Service に付与する任意のラベル |
| `defaultPort` | 3306番ポートの転送先を選ぶ。`mysql-rw` は書き込み可能なプライマリへ、`mysql-ro` はセカンダリへの読み取り専用接続へ、`mysql-rw-split` は Router の読み書きスプリット機能へ振り分ける |

以下は、Service を `LoadBalancer` として公開し、3306番ポートの既定の転送先を読み書きスプリットにする例である。

```yaml
# 以下は例である。実際のクラスタ名とアノテーション値は環境に合わせて変更する。
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  router:
    instances: 1
  service:
    type: LoadBalancer
    defaultPort: mysql-rw-split
```

動作確認は、更新後の Service に反映された `type` とポートの転送先を確認する。

```bash
kubectl get service mycluster -o jsonpath='{.spec.type}{"\n"}'
```

```text
LoadBalancer
```

## クライアントからの接続

アプリケーションからは、Router 向けの Service `mycluster` に対して、目的のポートで接続する。
ポート番号と用途の対応は[第11章 MySQL Router](11-mysql-router.md)で扱う。

```bash
kubectl run mysql-client --rm -it --image=mysql:8.4 --restart=Never -- \
  mysql -h mycluster -uroot -p
```

```text
Enter password:
Welcome to the MySQL monitor.
mysql>
```

## serviceFqdnTemplate による FQDN の制御

`spec.serviceFqdnTemplate` は、`<cluster名>-instances` の headless Service や個々の Pod が解決される FQDN のテンプレートを指定するフィールドである。

[crd.yaml の spec.serviceFqdnTemplate](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L860-L863)は次のとおりである。

```yaml
serviceFqdnTemplate:
  type: string
  description: "Template for a FQDN resolving to the cluster's headless instance Service and individual Pods"
  #default: "{service}.{namespace}.svc.{domain}" - We can't set the default as that would override the environment value from the operator
```

既定では `{service}.{namespace}.svc.{domain}` の形式で FQDN が組み立てられる。
クラスタが複数の Kubernetes クラスタにまたがる ClusterSet 構成など、既定のクラスタ内 DNS 名では到達できない環境では、`serviceFqdnTemplate` を使い、外部から解決可能なドメインのテンプレートに差し替える。

```yaml
# 以下は例である。domain の値は環境の DNS 設定に合わせる。
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  serviceFqdnTemplate: "{service}.{namespace}.svc.example.internal"
```

## トラブルシューティング

アプリケーションが `mycluster-instances` の個別 Pod に接続できない場合、まず Group Replication の状態を確認する。
readiness が揃っていない Pod にも到達できるのがこの Service の設計であり、`Not Ready` の Pod への接続がエラーになるときは、アプリケーション側ではなく Group Replication 側の復旧を先に確認する。

```bash
kubectl get pods -l mysql.oracle.com/cluster=mycluster
```

```text
NAME          READY   STATUS    RESTARTS   AGE
mycluster-0   2/2     Running   0          10m
mycluster-1   2/2     Running   0          10m
mycluster-2   1/2     Running   0          2m
```

## まとめ

InnoDB Cluster には、個々の MySQL Pod に直接到達する headless Service `<cluster名>-instances` と、Router を経由してアプリケーションが利用する Service `<cluster名>` の2種類がある。
アプリケーションからの通常の接続は後者を使い、`spec.service.defaultPort` で3306番ポートの転送先を選ぶ。

## 関連する章

- [第11章 MySQL Router](11-mysql-router.md)
- [第12章 TLS と証明書](12-tls.md)
