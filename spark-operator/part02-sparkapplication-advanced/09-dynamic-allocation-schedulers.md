# 第9章 動的リソース割り当てとバッチスケジューラ

> - [api/v1beta2/sparkapplication_types.go L144-L147](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L144-L147)
> - [api/v1beta2/sparkapplication_types.go L702-L726](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L702-L726)
> - [api/v1beta2/sparkapplication_types.go L126-L137](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L126-L137)
> - [examples/spark-pi-dynamic-allocation.yaml L16-L61](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-dynamic-allocation.yaml#L16-L61)
> - [examples/spark-pi-volcano.yaml L16-L57](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-volcano.yaml#L16-L57)
> - [examples/spark-pi-yunikorn.yaml L16-L59](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-yunikorn.yaml#L16-L59)
> - [examples/spark-pi-kube-scheduler.yaml L16-L57](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-kube-scheduler.yaml#L16-L57)

## この章でできるようになること

- **DynamicAllocation** を設定して、ワークロードに応じて executor 数を自動調整できる。
- **batchScheduler** で Volcano・YuniKorn・kube-scheduler を指定し、Spark Pod のスケジューリングを制御できる。

## 前提

- 第5章で executor の基本設定（`instances`・`cores`・`memory`）を理解していること。

## 動的リソース割り当て

**DynamicAllocation** は、ワークロードの負荷に応じて executor の数を動的に増減させる機能である。
Spark 3.0 以降で Kubernetes バックエンド向けに利用可能になった。

[api/v1beta2/sparkapplication_types.go の DynamicAllocation](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L702-L726)は次のように定義されている。

```go
// DynamicAllocation contains configuration options for dynamic allocation.
type DynamicAllocation struct {
	// Enabled controls whether dynamic allocation is enabled or not.
	Enabled bool `json:"enabled,omitempty"`
	// InitialExecutors is the initial number of executors to request. If .spec.executor.instances
	// is also set, the initial number of executors is set to the bigger of that and this option.
	// +optional
	InitialExecutors *int32 `json:"initialExecutors,omitempty"`
	// MinExecutors is the lower bound for the number of executors if dynamic allocation is enabled.
	// +optional
	MinExecutors *int32 `json:"minExecutors,omitempty"`
	// MaxExecutors is the upper bound for the number of executors if dynamic allocation is enabled.
	// +optional
	MaxExecutors *int32 `json:"maxExecutors,omitempty"`
	// ShuffleTrackingEnabled enables shuffle file tracking for executors, which allows dynamic allocation without
	// the need for an external shuffle service.
	// ...
	ShuffleTrackingEnabled *bool `json:"shuffleTrackingEnabled,omitempty"`
	// ShuffleTrackingTimeout controls the timeout in milliseconds for executors that are holding
	// shuffle data if shuffle tracking is enabled (true by default if dynamic allocation is enabled).
	// +optional
	ShuffleTrackingTimeout *int64 `json:"shuffleTrackingTimeout,omitempty"`
}
```

各フィールドの役割を次の表にまとめる。

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `enabled` | `bool` | 動的リソース割り当ての可否 |
| `initialExecutors` | `*int32` | 起動時の初期 executor 数 |
| `minExecutors` | `*int32` | executor 数の下限 |
| `maxExecutors` | `*int32` | executor 数の上限 |
| `shuffleTrackingEnabled` | `*bool` | Shuffle ファイル追跡の可否（デフォルト `true`） |
| `shuffleTrackingTimeout` | `*int64` | Shuffle データ保持 executor のタイムアウト（ミリ秒） |

`shuffleTrackingEnabled` を `true` にすると、外部 Shuffle Service を使わずに Shuffle ファイルを追跡できる。
この場合、Shuffle データを保持する executor はタイムアウトまで維持される。

### 動的リソース割り当ての例

[examples/spark-pi-dynamic-allocation.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-dynamic-allocation.yaml#L16-L61)は次のとおりである。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-dynamic-allocation
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  sparkVersion: 4.0.1
  driver:
    cores: 1
    memory: 512m
    serviceAccount: spark-operator-spark
  executor:
    instances: 1
    cores: 1
    memory: 512m
  dynamicAllocation:
    enabled: true
    initialExecutors: 2
    maxExecutors: 5
    minExecutors: 1
```

この設定では、起動時に2つの executor で開始し、ワークロードに応じて1つから5つの間で自動調整される。

## バッチスケジューラ

**batchScheduler** フィールドで、Spark Pod のスケジューリングに使用するスケジューラを指定できる。

[api/v1beta2/sparkapplication_types.go の batchScheduler](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L126-L137)の関連定義は次のとおりである。

```go
	// BatchScheduler configures which batch scheduler will be used for scheduling
	// +optional
	BatchScheduler *string `json:"batchScheduler,omitempty"`
	// BatchSchedulerOptions provides fine-grained control on how to batch scheduling.
	// +optional
	BatchSchedulerOptions *BatchSchedulerConfiguration `json:"batchSchedulerOptions,omitempty"`
```

サポートされるスケジューラを次の表にまとめる。

| スケジューラ名 | 説明 |
| --- | --- |
| `volcano` | Volcano バッチスケジューラ。Gang Scheduling やキュー管理を提供する |
| `yunikorn` | Apache YuniKorn スケジューラ。リソースキューとフェアシェアリングを提供する |
| `kube-scheduler` | Kubernetes のデフォルトスケジューラ。カスタムスケジューラ名を指定する場合に使用する |

### Volcano の例

[examples/spark-pi-volcano.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-volcano.yaml#L16-L57)は次のとおりである。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-volcano
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  sparkVersion: 4.0.1
  driver:
    cores: 1
    memory: 512m
    serviceAccount: spark-operator-spark
  executor:
    instances: 2
    cores: 1
    memory: 512m
  batchScheduler: volcano
```

`batchScheduler: volcano` を指定すると、Volcano が driver・executor Pod を Gang Scheduling でスケジューリングする。
Volcano を使用するには、クラスタに Volcano がインストールされている必要がある。

### YuniKorn の例

[examples/spark-pi-yunikorn.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-yunikorn.yaml#L16-L59)は次のとおりである。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-yunikorn
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  sparkVersion: 4.0.1
  driver:
    cores: 1
    memory: 512m
    serviceAccount: spark-operator-spark
  executor:
    instances: 2
    cores: 1
    memory: 512m
  batchScheduler: yunikorn
  batchSchedulerOptions:
    queue: root.default
```

YuniKorn では `batchSchedulerOptions.queue` でリソースキューを指定する。
`root.default` は YuniKorn のデフォルトキューである。

### kube-scheduler の例

[examples/spark-pi-kube-scheduler.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-kube-scheduler.yaml#L16-L57)は次のとおりである。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-kube-scheduler
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  sparkVersion: 4.0.1
  driver:
    cores: 1
    memory: 512m
    serviceAccount: spark-operator-spark
  executor:
    instances: 2
    cores: 1
    memory: 512m
  batchScheduler: kube-scheduler
```

`batchScheduler: kube-scheduler` を指定すると、Kubernetes のデフォルトスケジューラが使用される。
カスタムスケジューラ（Scheduling Framework を拡張したスケジューラ等）を使用する場合にも、このフィールドでスケジューラ名を指定する。

## Helm values でのバッチスケジューラの有効化

バッチスケジューラ機能は Helm values の `controller.batchScheduler.enable` で有効化する必要がある。

```yaml
controller:
  batchScheduler:
    enable: true
```

この設定を有効にしないと、`batchScheduler` フィールドを指定してもスケジューラは切り替わらない。

## まとめ

- **DynamicAllocation** で executor 数の自動調整を有効化できる。`minExecutors`・`maxExecutors` で範囲を指定する。
- **batchScheduler** で `volcano`・`yunikorn`・`kube-scheduler` を指定し、Spark Pod のスケジューリングを制御できる。
- バッチスケジューラ機能は Helm values の `controller.batchScheduler.enable: true` で有効化する。

## 関連する章

- [第5章 driver と executor の設定](../part01-sparkapplication-basics/05-driver-executor.md)
- [第14章 Helm values リファレンス](../part04-operations/14-helm-values.md)
