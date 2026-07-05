# 第12章 TLS と証明書

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L26-L38](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L26-L38)（`spec.secretName`、`spec.tlsCASecretName`、`spec.tlsSecretName`、`spec.tlsUseSelfSigned`）
> - [helm/mysql-operator/crds/crd.yaml#L281-L283](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L281-L283)（`spec.router.tlsSecretName`）
> - [mysqloperator/controller/innodbcluster/cluster_api.py#L1349-L1369](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_api.py#L1349-L1369)（証明書 Secret のマウント）
> - [mysqloperator/controller/innodbcluster/cluster_api.py#L1442-L1443](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_api.py#L1442-L1443)（`spec.router.tlsSecretName` の既定値）
> - [mysqloperator/controller/innodbcluster/cluster_api.py#L1882-L1889](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_api.py#L1882-L1889)（Router 証明書 Secret の読み込み）

## この章でできるようになること

InnoDB Cluster の通信を暗号化する2つの方式（自己署名証明書とユーザー提供証明書）を使い分けられるようになる。
証明書を保持する Secret を自分で用意し、`spec.tlsCASecretName` と `spec.tlsSecretName` で InnoDB Cluster に指定できるようになる。

## 前提

- [第5章 認証情報と Secret](../part01-innodbcluster-basics/05-credentials-secret.md)で Secret の基本を理解済みであること。

## TLS を制御するフィールド

InnoDB Cluster の TLS 関連フィールドは、`spec` の直下にまとまっている。

[crd.yaml の spec.secretName / tlsCASecretName / tlsSecretName / tlsUseSelfSigned](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L26-L38)は次のとおりである。

```yaml
secretName:
  type: string
  description: "Name of a generic type Secret containing root/default account password"
tlsCASecretName:
  type: string
  description: "Name of a generic type Secret containing CA (ca.pem) and optional CRL (crl.pem) for SSL"
tlsSecretName:
  type: string
  description: "Name of a TLS type Secret containing Server certificate and private key for SSL"
tlsUseSelfSigned:
  type: boolean
  default: false
  description: "Enables use of self-signed TLS certificates, reducing or disabling TLS based security verifications"
```

| フィールド | 説明 |
|---|---|
| `tlsCASecretName` | CA 証明書（`ca.pem`）と、任意で失効リスト（`crl.pem`）を格納した Secret の名前 |
| `tlsSecretName` | サーバー証明書と秘密鍵を格納した、Kubernetes の `kubernetes.io/tls` 型 Secret の名前 |
| `tlsUseSelfSigned` | 自己署名証明書を Operator に生成させるかどうか |

`tlsCASecretName` と `tlsSecretName` を省略すると、それぞれ `<cluster名>-ca`、`<cluster名>-tls` という名前の Secret を参照するとみなされる。
ただし `tlsUseSelfSigned` の既定値は `false` であるため、これらのフィールドを何も指定しないままにすると、Operator はこの2つの Secret が既に存在するという前提でクラスタを構築しようとする。
検証用にすぐ動かしたいだけであれば、次の「方式1」のように `tlsUseSelfSigned: true` を明示するほうが手間が少ない。

## 方式1: 自己署名証明書（`tlsUseSelfSigned: true`）

`tlsUseSelfSigned` を `true` にすると、Operator は証明書用の Secret を一切マウントせず、MySQL Server と MySQL Router がそれぞれ内部で自己署名証明書を自動生成する挙動に切り替わる。
このときの Group Replication のメンバー間通信は、証明書の身元検証を行わない `REQUIRED` モードで暗号化される。
証明書を自分で用意する必要がなく、検証環境やまず動かしてみたい場合に向く。

```yaml
# 以下は例である。secretName は事前に作成したパスワード用 Secret の名前に置き換える。
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  tlsUseSelfSigned: true
```

動作確認は、通信が TLS で暗号化されていることを `SHOW STATUS` で確認する。

```bash
kubectl exec mycluster-0 -c mysql -- \
  mysql -uroot -p -e "SHOW STATUS LIKE 'Ssl_cipher'"
```

```text
+---------------+---------------------------+
| Variable_name | Value                     |
+---------------+---------------------------+
| Ssl_cipher    | TLS_AES_256_GCM_SHA384    |
+---------------+---------------------------+
```

自己署名証明書は、証明書チェーンの検証を外部の認証局に委ねられない。
そのため、社外向けにエンドポイントを公開する構成や、コンプライアンス上の要件で正式な CA 署名を求められる環境には向かない。

## 方式2: ユーザー提供の証明書

正式な CA で署名した証明書を使う場合は、CA 証明書とサーバー証明書をあらかじめ Secret として作成し、`tlsCASecretName` と `tlsSecretName` で指定する。

CA 証明書用の Secret は `ca.pem` というキーで作成する。

```bash
# 以下は例である。ca.pem は事前に用意した CA 証明書ファイルに置き換える。
kubectl create secret generic mycluster-ca \
  --from-file=ca.pem=./ca.pem
```

サーバー証明書用の Secret は、Kubernetes の TLS 型 Secret として作成する。

```bash
# 以下は例である。tls.crt / tls.key は事前に用意したサーバー証明書と秘密鍵に置き換える。
kubectl create secret tls mycluster-server-tls \
  --cert=./server.crt \
  --key=./server.key
```

作成した Secret を InnoDBCluster から参照する。

```yaml
# 以下は例である。
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  tlsCASecretName: mycluster-ca
  tlsSecretName: mycluster-server-tls
```

動作確認は、Pod にマウントされた証明書ファイルを確認する。

```bash
kubectl exec mycluster-0 -c mysql -- ls /etc/mysql-ssl
```

```text
ca.pem
tls.crt
tls.key
```

cert-manager などの証明書管理ツールを運用している環境では、`Certificate` リソースの出力先 Secret 名をそのまま `tlsSecretName` に指定すれば、証明書の発行と更新は cert-manager 側に任せられる。
ただし CA 証明書は `ca.pem` というキー名で参照されるため、cert-manager が出力する `ca.crt` キーとは異なる。
cert-manager と組み合わせる場合は、`ca.crt` を `ca.pem` として複製する Secret をあわせて用意するか、CA 配布の仕組みを別途検討する必要がある。

## Router の TLS

MySQL Router のクライアント向け TLS は、`spec.router.tlsSecretName` で個別に指定できる。

[crd.yaml の spec.router.tlsSecretName](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L281-L283)は次のとおりである。

```yaml
tlsSecretName:
  type: string
  description: "Name of a TLS type Secret containing MySQL Router certificate and private key used for SSL"
```

[cluster_api.py の spec.router.tlsSecretName 既定値設定](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_api.py#L1442-L1443)は次のとおりである。

```python
if not self.router.tlsSecretName:
    self.router.tlsSecretName = f"{self.name}-router-tls"
```

省略した場合、Router は `<InnoDBCluster 名>-router-tls` という名前の Secret を既定値として参照する。
ただし、この Secret を Server 用の証明書のように事前に用意しておく必要はない。
[cluster_api.py の Router 証明書 Secret 読み込み](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_api.py#L1882-L1889)は、既定名の Secret が存在する場合だけ Router 用の証明書と秘密鍵を読み込む。

```python
try:
    router_tls_secret = cast(api_client.V1Secret, api_core.read_namespaced_secret(
                             self.parsed_spec.router.tlsSecretName, self.namespace))
    ret["router_tls.crt"] = utils.b64decode(router_tls_secret.data["tls.crt"])
    ret["router_tls.key"] = utils.b64decode(router_tls_secret.data["tls.key"])
except ApiException as e:
    if e.status != 404:
        raise
```

Secret が存在しなければ 404 を無視して処理を続け、Router 専用の証明書と秘密鍵は設定されない。
つまり省略時に Server と同じ証明書が使われるわけではなく、既定では Router 専用の証明書が設定されないだけである。
アプリケーションからの外部公開エンドポイントと、クラスタ内部のレプリケーション通信とで異なる証明書を使い分けたい場合や、Router にも証明書を持たせたい場合は、この項目で Secret 名を明示する。

```yaml
# 以下は例である。
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  tlsCASecretName: mycluster-ca
  tlsSecretName: mycluster-server-tls
  router:
    instances: 1
    tlsSecretName: mycluster-router-tls
```

## トラブルシューティング

TLS の Secret 名を誤って指定すると、Pod は Secret を待ち続けて起動しない。
`kubectl describe pod` の `Events` に `secret "..." not found` が出ていないかをまず確認する。

```bash
kubectl describe pod mycluster-0 | grep -A3 Events
```

```text
Events:
  Type     Reason       Message
  Warning  FailedMount  MountVolume.SetUp failed for volume "ssldata" : secret "mycluster-server-tls" not found
```

この場合、指定した Secret 名の誤りか、Secret 自体を作成し忘れていないかを確認する。

## まとめ

InnoDB Cluster の TLS は、`tlsUseSelfSigned` による自己署名証明書と、`tlsCASecretName` / `tlsSecretName` によるユーザー提供証明書のどちらかを選ぶ。
Router には `spec.router.tlsSecretName` で個別に証明書を指定できる。

## 関連する章

- [第10章 Service と接続エンドポイント](10-service-connection.md)
- [第11章 MySQL Router](11-mysql-router.md)
- [第5章 認証情報と Secret](../part01-innodbcluster-basics/05-credentials-secret.md)
