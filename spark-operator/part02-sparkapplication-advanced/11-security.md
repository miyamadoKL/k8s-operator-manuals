# 第11章 セキュリティと認証

> - [api/v1beta2/sparkapplication_types.go L49-L62](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L49-L62)
> - [api/v1beta2/sparkapplication_types.go L452-L460](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L452-L460)
> - [api/v1beta2/sparkapplication_types.go L596-L635](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L596-L635)
> - [charts/spark-operator-chart/values.yaml L156-L170](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L156-L170)
> - [charts/spark-operator-chart/values.yaml L461-L475](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L461-L475)

## この章でできるようになること

- ServiceAccount と RBAC の設定を理解し、Spark アプリケーションのアクセス制御を行える。
- Secret をマウントして、認証情報を安全にアプリケーションに渡せる。
- `proxyUser` を使ったユーザ代理（impersonation）を設定できる。

## 前提

- 第5章で driver・executor の基本設定を理解していること。
- 第6章で ConfigMap を Volume としてマウントする方法を経験していること。
- Kubernetes の RBAC（Role・RoleBinding・ServiceAccount）の基本概念を理解していること。

## ServiceAccount と RBAC

Spark Operator は Helm chart でインストール時に、controller 用と Spark アプリケーション用の2種類の ServiceAccount・RBAC を作成する。

### controller の ServiceAccount

[charts/spark-operator-chart/values.yaml の controller.serviceAccount](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L156-L164)は次のとおりである。

```yaml
  serviceAccount:
    # -- Specifies whether to create a service account for the controller.
    create: true
    # -- Optional name for the controller service account.
    name: ""
    # -- Extra annotations for the controller service account.
    annotations: {}
    # -- Auto-mount service account token to the controller pods.
    automountServiceAccountToken: true
```

[charts/spark-operator-chart/values.yaml の controller.rbac](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L166-L170)は次のとおりである。

```yaml
  rbac:
    # -- Specifies whether to create RBAC resources for the controller.
    create: true
    # -- Extra annotations for the controller RBAC resources.
    annotations: {}
```

controller の ServiceAccount には、SparkApplication の監視・更新、Pod の作成・削除、Service の作成等の権限が必要である。
`controller.rbac.create: true`（デフォルト）で、これらの権限を持つ ClusterRole・ClusterRoleBinding が自動作成される。

### Spark アプリケーションの ServiceAccount

[charts/spark-operator-chart/values.yaml の spark.serviceAccount](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L461-L469)は次のとおりである。

```yaml
  serviceAccount:
    # -- Specifies whether to create a service account for spark applications.
    create: true
    # -- Optional name for the spark service account.
    name: ""
    # -- Optional annotations for the spark service account.
    annotations: {}
    # -- Auto-mount service account token to the spark applications pods.
    automountServiceAccountToken: true
```

[charts/spark-operator-chart/values.yaml の spark.rbac](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L471-L475)は次のとおりである。

```yaml
  rbac:
    # -- Specifies whether to create RBAC resources for spark applications.
    create: true
    # -- Optional annotations for the spark application RBAC resources.
    annotations: {}
```

Spark アプリケーション用の ServiceAccount（デフォルト名 `spark-operator-spark`）は、driver Pod が executor Pod を管理するために使用される。
`spark.jobNamespaces` で指定された各 Namespace に作成される。

### driver での ServiceAccount 指定

`spec.driver.serviceAccount` で driver Pod が使用する ServiceAccount を指定する。

```yaml
spec:
  driver:
    serviceAccount: spark-operator-spark
```

この ServiceAccount には executor Pod の作成・削除・監視の権限が必要である。

## Secret のマウント

**SecretInfo** 構造体で、Secret を Pod にマウントする方法を指定できる。

[api/v1beta2/sparkapplication_types.go の SecretInfo](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L630-L635)は次のように定義されている。

```go
// SecretInfo captures information of a secret.
type SecretInfo struct {
	Name string     `json:"name"`
	Path string     `json:"path"`
	Type SecretType `json:"secretType"`
}
```

[api/v1beta2/sparkapplication_types.go の SecretType](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L603-L616)は次のように定義されている。

```go
// SecretType tells the type of a secret.
type SecretType string

// An enumeration of secret types supported.
const (
	// SecretTypeGCPServiceAccount is for secrets from a GCP service account Json key file that needs
	// the environment variable GOOGLE_APPLICATION_CREDENTIALS.
	SecretTypeGCPServiceAccount SecretType = "GCPServiceAccount"
	// SecretTypeHadoopDelegationToken is for secrets from an Hadoop delegation token that needs
	// the environment variable HADOOP_TOKEN_FILE_LOCATION.
	SecretTypeHadoopDelegationToken SecretType = "HadoopDelegationToken"
	// SecretTypeGeneric is for secrets that needs no special handling.
	SecretTypeGeneric SecretType = "Generic"
)
```

各 SecretType の用途を次の表にまとめる。

| SecretType | 説明 |
| --- | --- |
| `GCPServiceAccount` | GCP サービスアカウントの JSON キー。`GOOGLE_APPLICATION_CREDENTIALS` 環境変数が自動設定される |
| `HadoopDelegationToken` | Hadoop 委任トークン。`HADOOP_TOKEN_FILE_LOCATION` 環境変数が自動設定される |
| `Generic` | 特別な処理が不要な汎用 Secret |

以下は自作の設定例である。

```yaml
spec:
  driver:
    secrets:
    - name: gcp-sa-key
      path: /etc/secrets/gcp
      secretType: GCPServiceAccount
    - name: my-generic-secret
      path: /etc/secrets/generic
      secretType: Generic
```

`GCPServiceAccount` タイプを指定すると、`GOOGLE_APPLICATION_CREDENTIALS` 環境変数に `/etc/secrets/gcp/<key-file>` が自動設定される。

## Pod テンプレートでの Secret 参照

Pod テンプレートを使うと、ConfigMap・Secret からの環境変数注入をより柔軟に行える。

[api/v1beta2/sparkapplication_types.go の env](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L458-L460)の定義は次のとおりである。

```go
	// Env carries the environment variables to add to the pod.
	// +optional
	Env []corev1.EnvVar `json:"env,omitempty"`
```

以下は自作の設定例である。

```yaml
spec:
  driver:
    template:
      spec:
        containers:
        - name: spark-kubernetes-driver
          env:
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: db-credentials
                key: password
```

この設定では、`db-credentials` Secret の `password` キーを `DB_PASSWORD` 環境変数として注入する。

## proxyUser によるユーザ代理

**proxyUser** は `spark-submit` の `--proxy-user` フラグに相当し、アプリケーションを別のユーザとして実行するための設定である。

[api/v1beta2/sparkapplication_types.go の proxyUser](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L49-L52)の定義は次のとおりである。

```go
	// ProxyUser specifies the user to impersonate when submitting the application.
	// It maps to the command-line flag "--proxy-user" in spark-submit.
	// +optional
	ProxyUser *string `json:"proxyUser,omitempty"`
```

以下は自作の設定例である。

```yaml
spec:
  proxyUser: spark-user
```

`proxyUser` を指定すると、Spark アプリケーションは指定されたユーザとして実行される。
Hadoop/Spark の認証基盤（Kerberos 等）と連携して使用する。

## セキュリティコンテキスト

第5章で示したように、driver・executor の `securityContext` で Pod・コンテナのセキュリティ設定を指定できる。

以下は自作の設定例である。

```yaml
spec:
  driver:
    securityContext:
      runAsUser: 185
      runAsGroup: 185
      runAsNonRoot: true
    securityContext:
      capabilities:
        drop:
        - ALL
      allowPrivilegeEscalation: false
      seccompProfile:
        type: RuntimeDefault
```

`runAsNonRoot: true` で root 以外での実行を強制し、`capabilities.drop: [ALL]` で不要なケーパビリティを削除する。
これらの設定はセキュリティベストプラクティスとして推奨される。

## まとめ

- Spark Operator は controller 用と Spark アプリケーション用の ServiceAccount・RBAC を自動作成する。
- `secrets` フィールドで Secret をマウントできる。`GCPServiceAccount`・`HadoopDelegationToken` タイプでは環境変数が自動設定される。
- Pod テンプレートで `secretKeyRef` を使うと、Secret の値を環境変数として柔軟に注入できる。
- `proxyUser` で `spark-submit --proxy-user` 相当のユーザ代理を設定できる。
- `securityContext` で root 以外の実行、ケーパビリティの制限等を指定できる。

## 関連する章

- [第5章 driver と executor の設定](../part01-sparkapplication-basics/05-driver-executor.md)
- [第8章 Volume と Pod テンプレート](08-volumes-podtemplate.md)
- [第14章 Helm values リファレンス](../part04-operations/14-helm-values.md)
