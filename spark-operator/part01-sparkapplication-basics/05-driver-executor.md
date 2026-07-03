# 第5章 driver と executor の設定

> - [api/v1beta2/sparkapplication_types.go L419-L524](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L419-L524)
> - [api/v1beta2/sparkapplication_types.go L526-L565](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L526-L565)
> - [api/v1beta2/sparkapplication_types.go L567-L595](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L567-L595)
> - [examples/spark-pi-custom-resource.yaml L16-L60](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-custom-resource.yaml#L16-L60)

## この章でできるようになること

- **DriverSpec** と **ExecutorSpec** のフィールドを説明できる。
- **SparkPodSpec** が driver と executor に共通する設定項目を提供することを理解できる。
- CPU コアの要求量と上限、メモリ量、ラベル、環境変数、ServiceAccount を設定できる。

## 前提

- 第4章で SparkApplication の基本構造を理解していること。

## SparkPodSpec

**DriverSpec** と **ExecutorSpec** は、どちらも **SparkPodSpec** をインラインで埋め込んでいる。
SparkPodSpec が driver・executor に共通する設定項目を提供し、DriverSpec・ExecutorSpec がそれぞれの固有フィールドを追加する構造である。

[api/v1beta2/sparkapplication_types.go の SparkPodSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L419-L524)の主要なフィールドは次のとおりである。

```go
type SparkPodSpec struct {
	// Template is a pod template that can be used to define the driver or executor
	// pod configurations that Spark configurations do not support.
	// ...
	Template *corev1.PodTemplateSpec `json:"template,omitempty"`
	// Cores maps to `spark.driver.cores` or `spark.executor.cores` for the driver
	// and executors, respectively.
	// +optional
	// +kubebuilder:validation:Minimum=1
	Cores *int32 `json:"cores,omitempty"`
	// CoreLimit specifies a hard limit on CPU cores for the pod.
	// Optional
	CoreLimit *string `json:"coreLimit,omitempty"`
	// Memory is the amount of memory to request for the pod.
	// +optional
	Memory *string `json:"memory,omitempty"`
	// ...
	// Image is the container image to use. Overrides Spec.Image if set.
	// +optional
	Image *string `json:"image,omitempty"`
	// ...
	// Env carries the environment variables to add to the pod.
	// +optional
	Env []corev1.EnvVar `json:"env,omitempty"`
	// ...
	// Labels are the Kubernetes labels to be added to the pod.
	// +optional
	Labels map[string]string `json:"labels,omitempty"`
	// ...
	// ServiceAccount is the name of the custom Kubernetes service account used by the pod.
	// +optional
	ServiceAccount *string `json:"serviceAccount,omitempty"`
	// ...
}
```

共通フィールドの一覧を次の表にまとめる。

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `cores` | `*int32` | Spark に割り当てる CPU コア数（`spark.driver.cores` / `spark.executor.cores` に対応） |
| `coreLimit` | `*string` | CPU コアのハード上限（`spark.driver.coreLimit` / `spark.executor.coreLimit` に対応） |
| `memory` | `*string` | Pod が要求するメモリ量 |
| `memoryLimit` | `*string` | メモリのハード上限 |
| `memoryOverhead` | `*string` | ヒープ外のメモリ割り当て量（MiB 単位） |
| `image` | `*string` | コンテナイメージ（`spec.image` を上書き） |
| `env` | `[]corev1.EnvVar` | Pod に追加する環境変数 |
| `labels` | `map[string]string` | Pod に追加する Kubernetes ラベル |
| `annotations` | `map[string]string` | Pod に追加する Kubernetes アノテーション |
| `serviceAccount` | `*string` | Pod が使用する ServiceAccount 名 |
| `volumeMounts` | `[]corev1.VolumeMount` | Pod にマウントする Volume |
| `affinity` | `*corev1.Affinity` | Affinity・AntiAffinity の設定 |
| `tolerations` | `[]corev1.Toleration` | Taint の Toleration |
| `nodeSelector` | `map[string]string` | Node Selector |
| `securityContext` | `*corev1.SecurityContext` | コンテナのセキュリティコンテキスト |
| `template` | `*corev1.PodTemplateSpec` | Pod テンプレート（第8章で詳述） |

## DriverSpec の固有フィールド

[api/v1beta2/sparkapplication_types.go の DriverSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L526-L565)は SparkPodSpec に加えて次の固有フィールドを持つ。

```go
// DriverSpec is specification of the driver.
type DriverSpec struct {
	SparkPodSpec `json:",inline"`
	// PodName is the name of the driver pod that the user creates.
	// ...
	PodName *string `json:"podName,omitempty"`
	// CoreRequest is the physical CPU core request for the driver.
	// Maps to `spark.kubernetes.driver.request.cores` that is available since Spark 3.0.
	// +optional
	CoreRequest *string `json:"coreRequest,omitempty"`
	// JavaOptions is a string of extra JVM options to pass to the driver.
	// ...
	JavaOptions *string `json:"javaOptions,omitempty"`
	// ...
}
```

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `podName` | `*string` | `in-cluster-client` モードで使用する driver Pod の名前 |
| `coreRequest` | `*string` | 物理 CPU コアの要求量（`spark.kubernetes.driver.request.cores` に対応） |
| `javaOptions` | `*string` | driver に渡す追加の JVM オプション |
| `kubernetesMaster` | `*string` | driver が接続する Kubernetes マスタの URL（デフォルト `https://kubernetes.default.svc`） |
| `serviceAnnotations` | `map[string]string` | executor が driver に接続するための headless Service に追加するアノテーション |
| `ports` | `[]Port` | driver Pod のポート設定 |
| `priorityClassName` | `*string` | driver Pod の PriorityClass |

## ExecutorSpec の固有フィールド

[api/v1beta2/sparkapplication_types.go の ExecutorSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L567-L595)は SparkPodSpec に加えて次の固有フィールドを持つ。

```go
// ExecutorSpec is specification of the executor.
type ExecutorSpec struct {
	SparkPodSpec `json:",inline"`
	// Instances is the number of executor instances.
	// +optional
	// +kubebuilder:validation:Minimum=1
	Instances *int32 `json:"instances,omitempty"`
	// CoreRequest is the physical CPU core request for the executors.
	// Maps to `spark.kubernetes.executor.request.cores` that is available since Spark 2.4.
	// +optional
	CoreRequest *string `json:"coreRequest,omitempty"`
	// JavaOptions is a string of extra JVM options to pass to the executors.
	// ...
	JavaOptions *string `json:"javaOptions,omitempty"`
	// ...
	// DeleteOnTermination specify whether executor pods should be deleted in case
	// of failure or normal termination.
	// ...
	DeleteOnTermination *bool `json:"deleteOnTermination,omitempty"`
	// ...
}
```

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `instances` | `*int32` | executor のインスタンス数 |
| `coreRequest` | `*string` | 物理 CPU コアの要求量（`spark.kubernetes.executor.request.cores` に対応） |
| `javaOptions` | `*string` | executor に渡す追加の JVM オプション |
| `deleteOnTermination` | `*bool` | 終了時に executor Pod を削除するかどうか（`spark.kubernetes.executor.deleteOnTermination` に対応） |
| `ports` | `[]Port` | executor Pod のポート設定 |
| `priorityClassName` | `*string` | executor Pod の PriorityClass |

## cores と coreRequest の使い分け

`cores` は Spark 論理コア数を指定する。
`coreRequest` は Kubernetes 上の物理 CPU コアの要求量を指定する。
`cores` と `coreRequest` を併用すると、Spark には `cores` 分のタスクスロットを割り当てつつ、Kubernetes には `coreRequest` 分の CPU リソースだけを要求できる。
これにより、1つの executor Pod で複数のタスクを時間的に切り替えて実行する、いわゆる CPU オーバーサブスクリプションが可能になる。

## リソース指定の例

[examples/spark-pi-custom-resource.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-custom-resource.yaml#L16-L60)は `coreRequest` と `coreLimit` を使ったリソース指定の例である。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-custom-resource
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  sparkVersion: 4.0.1
  restartPolicy:
    type: Never
  driver:
    coreRequest: "0.5"
    coreLimit: 800m
    memory: 512m
    serviceAccount: spark-operator-spark
  executor:
    instances: 1
    coreRequest: "1200m"
    coreLimit: 1500m
    memory: 512m
```

driver は `coreRequest: "0.5"` で 0.5 コアを要求し、`coreLimit: 800m` で上限を 800 ミリコアに制限している。
executor は `coreRequest: "1200m"` で 1.2 コアを要求し、`coreLimit: 1500m` で上限を 1500 ミリコアに制限している。

## 動作確認

マニフェストを適用して、driver Pod のリソース要求を確認する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi-custom-resource.yaml
```

```text
sparkapplication.sparkoperator.k8s.io/spark-pi-custom-resource created
```

```bash
kubectl get pod spark-pi-custom-resource-driver -o jsonpath='{.spec.containers[0].resources}'
```

```text
{"limits":{"cpu":"800m","memory":"..."},"requests":{"cpu":"500m","memory":"..."}}
```

`requests.cpu` が `coreRequest` の値（`500m`）と一致し、`limits.cpu` が `coreLimit` の値（`800m`）と一致していることを確認できる。

## まとめ

- **SparkPodSpec** は driver・executor に共通する設定（`cores`・`memory`・`labels`・`env`・`serviceAccount` 等）を提供する。
- **DriverSpec** は `coreRequest`・`javaOptions`・`kubernetesMaster` 等の driver 固有フィールドを追加する。
- **ExecutorSpec** は `instances`・`coreRequest`・`deleteOnTermination` 等の executor 固有フィールドを追加する。
- `cores` は Spark 論理コア数、`coreRequest` は Kubernetes 物理 CPU 要求量を指定する。両者を併用することで CPU のオーバーサブスクリプションが可能になる。

## 関連する章

- [第4章 SparkApplication の基本構造](04-sparkapplication-spec.md)
- [第8章 Volume と Pod テンプレート](../part02-sparkapplication-advanced/08-volumes-podtemplate.md)
