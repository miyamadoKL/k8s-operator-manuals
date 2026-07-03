# 第4章 SparkApplication の基本構造

> - [api/v1beta2/sparkapplication_types.go L28-L71](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L28-L71)
> - [api/v1beta2/sparkapplication_types.go L211-L230](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L211-L230)
> - [examples/spark-pi.yaml L16-L62](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi.yaml#L16-L62)
> - [examples/spark-pi-python.yaml L16-L56](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-python.yaml#L16-L56)

## この章でできるようになること

- **SparkApplication** の `spec` に含まれる基本フィールドを説明できる。
- アプリケーションの言語（`type`）とデプロイモード（`mode`）を選択できる。
- Scala アプリケーションと Python アプリケーションのそれぞれのマニフェストを書ける。

## 前提

- 第1章で SparkApplication の概要を把握していること。
- 第3章で `kubectl apply` による SparkApplication の適用と状態確認を経験していること。

## SparkApplicationSpec の全体像

**SparkApplicationSpec** は `spark-submit` コマンドが受け取るすべての情報を表現する。

[api/v1beta2/sparkapplication_types.go の SparkApplicationSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L28-L71)の冒頭は次のように定義されている。

```go
// SparkApplicationSpec defines the desired state of SparkApplication
// It carries every pieces of information a spark-submit command takes and recognizes.
type SparkApplicationSpec struct {
	// ...

	// Type tells the type of the Spark application.
	// +kubebuilder:validation:Enum={Java,Python,Scala,R}
	Type SparkApplicationType `json:"type"`
	// SparkVersion is the version of Spark the application uses.
	SparkVersion string `json:"sparkVersion"`
	// Mode is the deployment mode of the Spark application.
	// +kubebuilder:validation:Enum={cluster,client}
	Mode DeployMode `json:"mode,omitempty"`
	// ...
	// Image is the container image for the driver, executor, and init-container.
	// ...
	Image *string `json:"image,omitempty"`
	// ...
	// MainClass is the fully-qualified main class of the Spark application.
	// This only applies to Java/Scala Spark applications.
	// +optional
	MainClass *string `json:"mainClass,omitempty"`
	// MainFile is the path to a bundled JAR, Python, or R file of the application.
	MainApplicationFile *string `json:"mainApplicationFile"`
	// Arguments is a list of arguments to be passed to the application.
	// +optional
	Arguments []string `json:"arguments,omitempty"`
}
```

基本フィールドの一覧を次の表にまとめる。

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `type` | `SparkApplicationType` | ○ | アプリケーションの言語。`Java`・`Scala`・`Python`・`R` のいずれか |
| `sparkVersion` | `string` | ○ | 使用する Spark のバージョン |
| `mode` | `DeployMode` | × | デプロイモード。`cluster`（デフォルト）または `client` |
| `image` | `*string` | × | driver・executor・init-container で使用するコンテナイメージ |
| `imagePullPolicy` | `*string` | × | イメージの取得ポリシー（`Always`・`IfNotPresent`・`Never`） |
| `mainClass` | `*string` | × | Java・Scala アプリケーションのエントリポイントとなるクラス |
| `mainApplicationFile` | `*string` | ○ | アプリケーションのファイルパス（JAR・Python・R） |
| `arguments` | `[]string` | × | アプリケーションに渡す引数 |

## アプリケーションの言語

[api/v1beta2/sparkapplication_types.go の SparkApplicationType](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L211-L220)は次のように定義されている。

```go
// SparkApplicationType describes the type of a Spark application.
type SparkApplicationType string

// Different types of Spark applications.
const (
	SparkApplicationTypeJava   SparkApplicationType = "Java"
	SparkApplicationTypeScala  SparkApplicationType = "Scala"
	SparkApplicationTypePython SparkApplicationType = "Python"
	SparkApplicationTypeR      SparkApplicationType = "R"
)
```

`type` フィールドには `Java`・`Scala`・`Python`・`R` のいずれかを指定する。
`Java` と `Scala` の場合は `mainClass` が必須になる。
`Python` と `R` の場合は `mainClass` を指定せず、`mainApplicationFile` にスクリプトファイルのパスを指定する。

## デプロイモード

[api/v1beta2/sparkapplication_types.go の DeployMode](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L222-L230)は次のように定義されている。

```go
// DeployMode describes the type of deployment of a Spark application.
type DeployMode string

// Different types of deployments.
const (
	DeployModeCluster         DeployMode = "cluster"
	DeployModeClient          DeployMode = "client"
	DeployModeInClusterClient DeployMode = "in-cluster-client"
)
```

| モード | 説明 |
| --- | --- |
| `cluster` | driver が Kubernetes クラスタ内の Pod として実行される。バッチジョブではこのモードが標準 |
| `client` | driver がクラスタ外で実行され、executor だけがクラスタ内で起動する |
| `in-cluster-client` | driver をユーザが作成した Pod 内で実行する特殊なモード |

バッチ処理では `cluster` モードを使用するのが一般的である。

## Scala アプリケーションの例

[examples/spark-pi.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi.yaml#L16-L62)は Scala による SparkApplication の例である。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  arguments:
  - "5000"
  sparkVersion: 4.0.1
  # ... (driver, executor の設定は省略) ...
```

`type: Scala` を指定し、`mainClass` にエントリポイント、`mainApplicationFile` に JAR ファイルのパスを指定する。
`local://` スキームは、コンテナイメージ内に含まれるファイルを参照する。

## Python アプリケーションの例

[examples/spark-pi-python.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-python.yaml#L16-L56)は Python による SparkApplication の例である。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-python
  namespace: default
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainApplicationFile: local:///opt/spark/examples/src/main/python/pi.py
  sparkVersion: 4.0.1
  # ... (driver, executor の設定は省略) ...
```

Python アプリケーションでは `type: Python` を指定し、`mainClass` は記述しない。
`mainApplicationFile` に Python スクリプトのパスを指定する。
`pythonVersion` で Python のメジャーバージョン（`"2"` または `"3"`）を選択できる。

## 動作確認

Scala 版の `spark-pi` を適用して状態を確認する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi.yaml
```

```text
sparkapplication.sparkoperator.k8s.io/spark-pi created
```

```bash
kubectl get sparkapp spark-pi
```

```text
NAME       STATUS      ATTEMPTS   START              FINISH             AGE
spark-pi   COMPLETED   1          2026-01-01 00:00   2026-01-01 00:01   2m
```

Python 版も同様に適用できる。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi-python.yaml
```

```text
sparkapplication.sparkoperator.k8s.io/spark-pi-python created
```

## まとめ

- `SparkApplicationSpec` は `spark-submit` の全パラメータを表現する。
- `type` で言語（`Java`・`Scala`・`Python`・`R`）を、`mode` でデプロイモード（`cluster`・`client`）を指定する。
- Java・Scala は `mainClass` と `mainApplicationFile` を、Python・R は `mainApplicationFile` だけを指定する。

## 関連する章

- [第3章 クイックスタート](../part00-introduction/03-quickstart.md)
- [第5章 driver と executor の設定](05-driver-executor.md)
