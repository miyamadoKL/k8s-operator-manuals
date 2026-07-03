# 第13章 SparkConnect

> - [api/v1alpha1/sparkconnect_types.go L36-L52](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L36-L52)
> - [api/v1alpha1/sparkconnect_types.go L54-L85](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L54-L85)
> - [api/v1alpha1/sparkconnect_types.go L87-L125](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L87-L125)
> - [api/v1alpha1/sparkconnect_types.go L127-L191](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L127-L191)
> - [config/crd/bases/sparkoperator.k8s.io_sparkconnects.yaml L1-L17](https://github.com/kubeflow/spark-operator/blob/v2.5.1/config/crd/bases/sparkoperator.k8s.io_sparkconnects.yaml#L1-L17)
> - [examples/sparkconnect/spark-connect.yaml L16-L81](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/sparkconnect/spark-connect.yaml#L16-L81)

## この章でできるようになること

- **SparkConnect** の目的と SparkApplication との違いを理解できる。
- SparkConnect の spec 構造を説明できる。
- Spark Connect サーバへの接続方法を理解できる。

## 前提

- 第4章で SparkApplication の基本構造を理解していること。
- Spark Connect の基本概念（対話型セッション、リモート接続）を理解していること。

## SparkConnect の概要

**SparkConnect** は Spark Connect サーバを Kubernetes 上で起動し、長時間接続型の対話セッションを提供する Custom Resource である。
API バージョンは `v1alpha1` であり、`v1beta2` の SparkApplication とは独立した型である。

[config/crd/bases/sparkoperator.k8s.io_sparkconnects.yaml の CRD 定義](https://github.com/kubeflow/spark-operator/blob/v2.5.1/config/crd/bases/sparkoperator.k8s.io_sparkconnects.yaml#L1-L17)は次のとおりである。

```yaml
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    api-approved.kubernetes.io: https://github.com/kubeflow/spark-operator/pull/1298
    controller-gen.kubebuilder.io/version: v0.17.1
  name: sparkconnects.sparkoperator.k8s.io
spec:
  group: sparkoperator.k8s.io
  names:
    kind: SparkConnect
    listKind: SparkConnectList
    plural: sparkconnects
    shortNames:
    - sparkconn
    singular: sparkconnect
```

ショートネームは `sparkconn` であり、`kubectl get sparkconn` で状態を確認できる。

## SparkConnect と SparkApplication の違い

**SparkConnect** と **SparkApplication** は、どちらも Spark ワークロードを Kubernetes 上で実行するための Custom Resource だが、その目的と動作は異なる。

| 項目 | SparkApplication | SparkConnect |
| --- | --- | --- |
| 目的 | バッチジョブの実行 | 対話型セッションの提供 |
| API バージョン | `v1beta2` | `v1alpha1` |
| 実行モデル | 一度きりのジョブ（完了後に終了） | 長時間稼働するサーバ |
| 接続方法 | なし（ジョブが自律的に実行） | クライアントがリモート接続 |
| 用途 | ETL、データ処理パイプライン | Jupyter Notebook、データ分析 |

**SparkApplication** は `spark-submit` を自動化し、ジョブが完了すると Pod が終了する。
**SparkConnect** は Spark Connect サーバを起動し、クライアント（PySpark、Spark SQL CLI 等）がリモート接続して対話的にクエリを実行できる。

## SparkConnectSpec

[api/v1alpha1/sparkconnect_types.go の SparkConnectSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L54-L85)は次のように定義されている。

```go
// SparkConnectSpec defines the desired state of SparkConnect.
type SparkConnectSpec struct {
	// SparkVersion is the version of Spark the spark connect use.
	SparkVersion string `json:"sparkVersion"`

	// Image is the container image for the driver, executor, and init-container.
	// ...
	Image *string `json:"image,omitempty"`

	// HadoopConf carries user-specified Hadoop configuration properties...
	// ...
	HadoopConf map[string]string `json:"hadoopConf,omitempty"`

	// SparkConf carries user-specified Spark configuration properties...
	// ...
	SparkConf map[string]string `json:"sparkConf,omitempty"`

	// Server is the Spark connect server specification.
	Server ServerSpec `json:"server"`

	// Executor is the Spark executor specification.
	Executor ExecutorSpec `json:"executor"`

	// DynamicAllocation configures dynamic allocation...
	// ...
	DynamicAllocation *DynamicAllocation `json:"dynamicAllocation,omitempty"`
}
```

各フィールドの役割を次の表にまとめる。

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `sparkVersion` | `string` | 使用する Spark のバージョン |
| `image` | `*string` | サーバ・executor で使用するコンテナイメージ |
| `hadoopConf` | `map[string]string` | Hadoop 設定プロパティ |
| `sparkConf` | `map[string]string` | Spark 設定プロパティ |
| `server` | `ServerSpec` | Spark Connect サーバの仕様 |
| `executor` | `ExecutorSpec` | executor の仕様 |
| `dynamicAllocation` | `*DynamicAllocation` | 動的リソース割り当ての設定 |

## ServerSpec と ExecutorSpec

**ServerSpec** は Spark Connect サーバの仕様を定義する。

[api/v1alpha1/sparkconnect_types.go の ServerSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L87-L94)は次のように定義されている。

```go
// ServerSpec is specification of the Spark connect server.
type ServerSpec struct {
	SparkPodSpec `json:",inline"`

	// Service exposes the Spark connect server.
	// +optional
	Service *corev1.Service `json:"service,omitempty"`
}
```

**ExecutorSpec** は executor の仕様を定義する。

[api/v1alpha1/sparkconnect_types.go の ExecutorSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L96-L104)は次のように定義されている。

```go
// ExecutorSpec is specification of the executor.
type ExecutorSpec struct {
	SparkPodSpec `json:",inline"`

	// Instances is the number of executor instances.
	// +optional
	// +kubebuilder:validation:Minimum=0
	Instances *int32 `json:"instances,omitempty"`
}
```

**SparkPodSpec** はサーバ・executor に共通する設定項目を提供する。

[api/v1alpha1/sparkconnect_types.go の SparkPodSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L106-L125)は次のように定義されている。

```go
// SparkPodSpec defines common things that can be customized for a Spark driver or executor pod.
type SparkPodSpec struct {
	// Cores maps to `spark.driver.cores` or `spark.executor.cores` for the driver and executors, respectively.
	// +optional
	// +kubebuilder:validation:Minimum=1
	Cores *int32 `json:"cores,omitempty"`

	// Memory is the amount of memory to request for the pod.
	// +optional
	Memory *string `json:"memory,omitempty"`

	// Template is a pod template that can be used to define the driver or executor pod configurations...
	// ...
	Template *corev1.PodTemplateSpec `json:"template,omitempty"`
}
```

## SparkConnectStatus

[api/v1alpha1/sparkconnect_types.go の SparkConnectStatus](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L127-L151)は次のように定義されている。

```go
// SparkConnectStatus defines the observed state of SparkConnect.
type SparkConnectStatus struct {
	// Conditions represents the latest available observations of a SparkConnect's current state.
	Conditions []metav1.Condition `json:"conditions,omitempty" ...`

	// State represents the current state of the SparkConnect.
	State SparkConnectState `json:"state,omitempty"`

	// Server represents the current state of the SparkConnect server.
	Server SparkConnectServerStatus `json:"server,omitempty"`

	// Executors represents the current state of the SparkConnect executors.
	Executors map[string]int `json:"executors,omitempty"`

	// StartTime is the time at which the SparkConnect controller started processing...
	StartTime metav1.Time `json:"startTime,omitempty"`

	// LastUpdateTime is the time at which the SparkConnect controller last updated...
	LastUpdateTime metav1.Time `json:"lastUpdateTime,omitempty"`
}
```

[api/v1alpha1/sparkconnect_types.go の SparkConnectState](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1alpha1/sparkconnect_types.go#L170-L180)は次のように定義されている。

```go
// SparkConnectState represents the current state of the SparkConnect.
type SparkConnectState string

// All possible states of the SparkConnect.
const (
	SparkConnectStateNew          SparkConnectState = ""
	SparkConnectStateProvisioning SparkConnectState = "Provisioning"
	SparkConnectStateReady        SparkConnectState = "Ready"
	SparkConnectStateNotReady     SparkConnectState = "NotReady"
	SparkConnectStateFailed       SparkConnectState = "Failed"
)
```

状態遷移を次の表にまとめる。

| 状態 | 説明 |
| --- | --- |
| `New` | Custom Resource が作成された初期状態 |
| `Provisioning` | サーバ Pod の起動を進めている |
| `Ready` | サーバが稼働し、クライアントからの接続を受け付けられる |
| `NotReady` | サーバが一時的に利用できない |
| `Failed` | サーバの起動に失敗した |

## 実行例

[examples/sparkconnect/spark-connect.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/sparkconnect/spark-connect.yaml#L16-L81)は次のとおりである。

```yaml
apiVersion: sparkoperator.k8s.io/v1alpha1
kind: SparkConnect
metadata:
  name: spark-connect
  namespace: default
spec:
  sparkVersion: 4.0.1
  server:
    template:
      metadata:
        labels:
          key1: value1
      spec:
        containers:
        - name: spark-kubernetes-driver
          image: docker.io/library/spark:4.0.1
          imagePullPolicy: Always
          resources:
            requests:
              cpu: 1
              memory: 1Gi
            limits:
              cpu: 1
              memory: 1Gi
        serviceAccount: spark-operator-spark
  executor:
    instances: 2
    cores: 1
    memory: 512m
    template:
      spec:
        containers:
        - name: spark-kubernetes-executor
          image: docker.io/library/spark:4.0.1
          imagePullPolicy: Always
```

この設定では、1コア・1GiB メモリの Spark Connect サーバが起動し、2つの executor（各1コア・512MiB）が作成される。

## 動作確認

SparkConnect を適用する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/sparkconnect/spark-connect.yaml
```

```text
sparkconnect.sparkoperator.k8s.io/spark-connect created
```

状態を確認する。

```bash
kubectl get sparkconn spark-connect
```

```text
NAME            STATUS    PODNAME              AGE
spark-connect   Ready     spark-connect-server 1m
```

サーバが `Ready` 状態になったら、Service を経由して接続できる。

```bash
kubectl get service spark-connect-server
```

```text
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
spark-connect-server ClusterIP   10.96.xxx.xxx   <none>        15002/TCP  1m
```

## Spark Connect サーバへの接続

Spark Connect サーバへの接続は、PySpark の `SparkSession.builder.remote()` を使用する。

```python
from pyspark.sql import SparkSession

# クラスタ内から接続する場合
spark = SparkSession.builder \
    .remote("sc://spark-connect-server.default.svc:15002") \
    .getOrCreate()

# ローカルから port-forward 経由で接続する場合
# kubectl port-forward svc/spark-connect-server 15002:15002
spark = SparkSession.builder \
    .remote("sc://localhost:15002") \
    .getOrCreate()
```

接続先は `sc://<host>:<port>` 形式の URI で指定する。
デフォルトのポートは `15002` である。

## まとめ

- **SparkConnect** は Spark Connect サーバを Kubernetes 上で起動し、対話型セッションを提供する。
- API バージョンは `v1alpha1` であり、バッチジョブ用の `v1beta2` SparkApplication とは独立している。
- `server` フィールドでサーバの設定を、`executor` フィールドで executor の設定を指定する。
- サーバが `Ready` 状態になったら、`sc://<host>:<port>` 形式の URI で PySpark から接続できる。

## 関連する章

- [第1章 Spark Operator とは](../part00-introduction/01-what-is-spark-operator.md)
- [第4章 SparkApplication の基本構造](../part01-sparkapplication-basics/04-sparkapplication-spec.md)
