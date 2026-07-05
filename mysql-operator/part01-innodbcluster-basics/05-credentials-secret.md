# 第5章 認証情報と Secret

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L26-L28](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L26-L28)
> - [samples/sample-secret.yaml#L1-L19](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-secret.yaml#L1-L19)
> - [mysqloperator/controller/config.py#L58-L60](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/config.py#L58-L60)
> - [mysqloperator/controller/innodbcluster/cluster_objects.py#L75-L90](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L75-L90)
> - [mysqloperator/sidecar_main.py#L242-L259](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/sidecar_main.py#L242-L259)

## この章でできるようになること

InnoDBCluster が root ユーザーの認証情報をどの Secret から読むかを理解し、その Secret を自分で用意して `secretName` に指定できるようになる。
あわせて、Operator がクラスタ運用のために内部で作成する Secret とユーザーの位置づけを説明できるようになる。

## 前提

第4章で InnoDBCluster の `spec` 全体像を確認済みであることを前提とする。

## secretName フィールド

InnoDBCluster の `spec` で唯一の必須フィールドが `secretName` である。

[helm/mysql-operator/crds/crd.yaml#L26-L28](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L26-L28)

```yaml
                secretName:
                  type: string
                  description: "Name of a generic type Secret containing root/default account password"
```

`secretName` には、root ユーザー（または任意の名前を付けたデフォルトユーザー）の認証情報を持つ `Secret` の名前を指定する。
この Secret は InnoDBCluster と同じ namespace に、Operator を適用する前にあらかじめ作成しておく必要がある。

## Secret の作り方

サイドカーコンテナが起動時にこの Secret を読み、root アカウントの情報を組み立てる。

[mysqloperator/sidecar_main.py#L242-L259](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/sidecar_main.py#L242-L259)

```python
def get_root_account_info(cluster: InnoDBCluster) -> Tuple[str, str, str]:
    secrets = cluster.get_user_secrets()
    if secrets:
        user = secrets.data.get("rootUser")
        host = secrets.data.get("rootHost")
        password = secrets.data.get("rootPassword", None)
        if not password:
            raise Exception(
                f"rootPassword missing in secret {cluster.parsed_spec.secretName}")
        if user:
            user = utils.b64decode(user)
        else:
            user = "root"
        if host:
            host = utils.b64decode(host)
        else:
            host = "%"
        password = utils.b64decode(password)
```

この実装からわかる通り、Secret に必須のキーは `rootPassword` だけである。
`rootUser` を省略すると既定値 `root` が、`rootHost` を省略すると既定値 `%`（すべてのホストからの接続を許可）が使われる。
サンプルや以降の例では説明のために3キーを明示するが、`rootUser` と `rootHost` は省略してもクラスタは初期化できる。
公式サンプルは次の通りである。

[samples/sample-secret.yaml#L12-L19](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-secret.yaml#L12-L19)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mypwds
stringData:
  rootUser: replace me with a username like root
  rootHost: '%'
  rootPassword: set me to a password
```

- `rootUser`：作成するユーザー名。典型的には `root` だが、任意の名前を選べる。
- `rootHost`：このユーザーが接続を許可されるホストパターン。`%` はすべてのホストからの接続を許可する。
- `rootPassword`：パスワード。

以下は自分の環境向けに値を差し替えた例である。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mypwds
stringData:
  rootUser: root
  rootHost: '%'
  rootPassword: <実際のパスワードに置き換える>
```

動作確認として、Secret を適用してからキーが揃っているかを確認する。

```bash
kubectl apply -f secret.yaml
kubectl get secret mypwds -o jsonpath='{.data}' | jq 'keys'
```

```text
["rootHost", "rootPassword", "rootUser"]
```

`rootPassword` が欠けている場合、InnoDBCluster の作成時に Operator がエラーを報告し、クラスタの初期化が進まない。
`rootUser` や `rootHost` が欠けている場合は、それぞれ既定値 `root` と `%` にフォールバックするため、初期化は進む。

## InnoDBCluster からの参照

作成した Secret 名を `spec.secretName` に指定する。

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  router:
    instances: 1
```

Operator はこの Secret から `rootUser`（省略時は `root`）を読み取り、`rootHost`（省略時は `%`）を許可ホストとして、クラスタ初期化時に該当ユーザーを作成する。

## Operator が内部で作成する Secret とユーザー

`secretName` で指定した Secret は、利用者が用意する唯一の認証情報だが、Operator はクラスタ運用のためにさらにいくつかの内部ユーザーを自動作成する。

[mysqloperator/controller/config.py#L58-L60](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/config.py#L58-L60)

```python
CLUSTER_ADMIN_USER_NAME = "mysqladmin"
ROUTER_METADATA_USER_NAME = "mysqlrouter"
BACKUP_USER_NAME = "mysqlbackup"
```

- `mysqladmin`：Operator 自身がクラスタの管理操作（Group Replication の構成変更など）に使う内部ユーザーである。
- `mysqlrouter`：MySQL Router がメタデータ参照に使う内部ユーザーである（第11章で扱う）。
- `mysqlbackup`：バックアップ実行時に使う内部ユーザーである（第15章で扱う）。

これらのユーザーに対応する Secret は、Operator がクラスタ初期化時に自動生成し、利用者が個別に用意する必要はない。
たとえば `mysqladmin` の認証情報は、次のように `<クラスタ名>-privsecrets` という名前の Secret として生成される。

[mysqloperator/controller/innodbcluster/cluster_objects.py#L82-L90](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L82-L90)

```python
    tmpl = f"""
apiVersion: v1
kind: Secret
metadata:
  name: {spec.name}-privsecrets
data:
  clusterAdminUsername: {admin_user}
  clusterAdminPassword: {admin_pwd}
"""
```

利用者が意識すべき Secret は `secretName` で指定するものだけである。

動作確認として、Operator が作成した内部 Secret を確認する。

```bash
kubectl get secret mycluster-privsecrets -o jsonpath='{.data}' | jq 'keys'
```

```text
["clusterAdminPassword", "clusterAdminUsername"]
```

`<クラスタ名>-privsecrets` という名前の Secret に、`mysqladmin` ユーザーの認証情報が Operator によって自動生成されていることが確認できる。
利用者が用意するのは `secretName` で指定した Secret だけであり、この `-privsecrets` Secret を自分で作る必要はない。

## まとめ

InnoDBCluster に必須の情報は、root ユーザーの認証情報を持つ Secret の名前（`secretName`）だけである。
Secret のキーのうち実装上必須なのは `rootPassword` のみであり、`rootUser`（既定 `root`）と `rootHost`（既定 `%`）は省略できる。
サンプルでは説明のために `rootUser`、`rootHost`、`rootPassword` の3キーを用意する。
Operator は、この Secret とは別に `mysqladmin`、`mysqlrouter`、`mysqlbackup` という内部ユーザーとその Secret を自動生成し、クラスタ運用に利用する。

## 関連する章

- 前章：[第4章 InnoDBCluster リソースの全体像](04-innodbcluster-resource.md)
- 次章：[第6章 サーバー構成](06-server-config.md)
- [第11章 MySQL Router](../part02-networking/11-mysql-router.md)
- [第15章 バックアップの概念とプロファイル](../part04-backup-restore/15-backup-concepts.md)
