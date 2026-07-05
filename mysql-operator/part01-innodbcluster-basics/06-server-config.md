# 第6章 サーバー構成

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L39-L80](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L39-L80)
> - [mysqloperator/controller/config.py#L23-L52](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/config.py#L23-L52)

## この章でできるようになること

InnoDBCluster の `instances`、`version`、`edition`、`baseServerId`、`imageRepository`、`imagePullPolicy` を指定して、クラスタの規模とサーバーイメージを制御できるようになる。

## 前提

第5章で `secretName` を用意済みであることを前提とする。

## instances

`instances` は、クラスタを構成する MySQL Server の数である。

[helm/mysql-operator/crds/crd.yaml#L75-L80](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L75-L80)

```yaml
                instances:
                  type: integer
                  minimum: 1
                  maximum: 9
                  default: 1
                  description: "Number of MySQL replica instances for the cluster"
```

1から9までの整数を指定できる。
省略時は1（単一インスタンス）になる。
3以上の奇数を指定すると、Group Replication の多数決による自動フェイルオーバーが機能する。
2インスタンスの構成では多数決が成立せず、片系が落ちるとクラスタ全体が書き込み不能になるため、本番運用では3以上の奇数を推奨する。

## version と edition

MySQL Server のバージョンと Edition は次の2フィールドで指定する。

[helm/mysql-operator/crds/crd.yaml#L39-L46](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L39-L46)

```yaml
                version:
                  type: string
                  pattern: '^\d+\.\d+\.\d+(-.+)?'
                  description: "MySQL Server version"
                edition:
                  type: string
                  pattern: "^(community|enterprise)$"
                  description: "MySQL Server Edition (community or enterprise)"
```

`edition` は `community` か `enterprise` のいずれかに限定される。
`version` を省略した場合のデフォルト値は、Operator のビルド定数から決まる。

[mysqloperator/controller/config.py#L24-L32](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/config.py#L24-L32)

```python
OPERATOR_EDITION = Edition.community

SHELL_VERSION = "8.4.9"

DEFAULT_VERSION_TAG = "8.4.9"
```

この対象バージョンの Operator（`8.4.9-2.1.11`）では、`version` 省略時に `8.4.9` が、`edition` 省略時に `community` が使われる。
本書は community 版を主軸に解説するため、以後のマニフェスト例で `edition` は省略する。

## baseServerId

Group Replication では、クラスタ内の各インスタンスに一意な `server_id` を割り当てる必要がある。
`baseServerId` は、その採番の基準値である。

[helm/mysql-operator/crds/crd.yaml#L62-L67](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L62-L67)

```yaml
                baseServerId:
                  type: integer
                  minimum: 0
                  maximum: 4294967195
                  default: 1000
                  description: "Base value for MySQL server_id for instances in the cluster"
```

Operator は `baseServerId` を起点に、インスタンスごとに連番の `server_id` を割り当てる。
同一 Kubernetes クラスタ内で複数の InnoDBCluster を運用する場合、`server_id` の採番範囲が重ならないよう、クラスタごとに異なる `baseServerId` を指定する必要がある。

## imageRepository と imagePullPolicy

[helm/mysql-operator/crds/crd.yaml#L47-L52](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L47-L52)

```yaml
                imageRepository:
                  type: string
                  description: "Repository where images are pulled from; defaults to container-registry.oracle.com/mysql"
                imagePullPolicy:
                  type: string
                  description: "Defaults to Always, but set to IfNotPresent in deploy-operator.yaml when deploying Operator"
```

`imageRepository` を省略すると、Operator の設定定数から取得される。

[mysqloperator/controller/config.py#L47-L52](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/config.py#L47-L52)

```python
DEFAULT_IMAGE_REPOSITORY = os.getenv(
    "MYSQL_OPERATOR_DEFAULT_REPOSITORY", default="container-registry.oracle.com/mysql").rstrip('/')

MYSQL_SERVER_IMAGE = "community-server"
```

`imagePullPolicy` は省略時 `Always` になるが、この記述は crd.yaml の説明文にある通り、`deploy/deploy-operator.yaml` で Operator 自身をデプロイする際には `IfNotPresent` に設定されている。
イメージレジストリへのアクセスを制限された環境では、`imageRepository` を社内レジストリに向け、`imagePullSecrets` と組み合わせて使う。

## マニフェスト例

以下は3インスタンス構成で、バージョンと `baseServerId` を明示した例である。

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  baseServerId: 2000
  version: 8.4.9
  router:
    instances: 1
```

動作確認として、生成された各 Pod の `server_id` を確認する。

```bash
kubectl exec mycluster-0 -c mysql -- mysql -uroot -p"$(kubectl get secret mypwds -o jsonpath='{.data.rootPassword}' | base64 -d)" -N -e "SELECT @@server_id"
```

```text
2000
```

`mycluster-0` の `server_id` が `baseServerId` と一致することが確認できる。
`mycluster-1`、`mycluster-2` はそれぞれ `2001`、`2002` になる。

## トラブルシューティング

`instances` を偶数に設定して運用し、片系が停止すると、残りのインスタンス数によっては Group Replication の多数決が成立せず、クラスタ全体が読み取り専用になる場合がある。
`kubectl get innodbcluster` の `STATUS` 列が `ONLINE` から変化した場合は、まず `instances` の構成と現在の稼働インスタンス数を確認する。

## まとめ

`instances`、`version`、`edition`、`baseServerId`、`imageRepository`、`imagePullPolicy` は、クラスタの規模とサーバーイメージを決める基本フィールドである。
`instances` は3以上の奇数を選ぶことで、多数決による自動フェイルオーバーが機能する。
`version` と `edition` を省略すると、Operator のビルド時点のデフォルト（`8.4.9` / `community`）が使われる。

## 関連する章

- 前章：[第5章 認証情報と Secret](05-credentials-secret.md)
- 次章：[第7章 ストレージと PersistentVolumeClaim](07-storage-pvc.md)
- [第19章 スケーリングとアップグレード](../part05-operations/19-scaling-upgrade.md)
