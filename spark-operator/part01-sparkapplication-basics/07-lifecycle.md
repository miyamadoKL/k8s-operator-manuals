# 第7章 ライフサイクルと再実行

> - [api/v1beta2/sparkapplication_types.go L232-L268](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L232-L268)
> - [api/v1beta2/sparkapplication_types.go L338-L357](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L338-L357)
> - [api/v1beta2/sparkapplication_types.go L129-L134](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L129-L134)
> - [examples/spark-pi-ttl.yaml L16-L57](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-ttl.yaml#L16-L57)

## この章でできるようになること

- **RestartPolicy** を設定して、失敗したアプリケーションの自動再実行を制御できる。
- **timeToLiveSeconds** で完了後の SparkApplication の自動削除を設定できる。
- アプリケーションの状態遷移（状態機械）を読み取り、現在の状態から次の動作を判断できる。

## 前提

- 第1章で SparkApplication の状態遷移の全体像を把握していること。
- 第4章で SparkApplication の基本構造を理解していること。

## RestartPolicy

**RestartPolicy** はアプリケーションの終了後に再実行するかどうかを制御する。

[api/v1beta2/sparkapplication_types.go の RestartPolicy](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L232-L268)は次のように定義されている。

```go
// RestartPolicy is the policy of if and in which conditions the controller should restart a terminated application.
// This completely defines actions to be taken on any kind of Failures during an application run.
type RestartPolicy struct {
	// Type specifies the RestartPolicyType.
	// +kubebuilder:validation:Enum={Never,Always,OnFailure}
	Type RestartPolicyType `json:"type,omitempty"`

	// OnSubmissionFailureRetries is the number of times to retry submitting an application before giving up.
	// ...
	OnSubmissionFailureRetries *int32 `json:"onSubmissionFailureRetries,omitempty"`

	// OnFailureRetries the number of times to retry running an application before giving up.
	// +optional
	OnFailureRetries *int32 `json:"onFailureRetries,omitempty"`

	// OnSubmissionFailureRetryInterval is the interval in seconds between retries on failed submissions.
	// +optional
	OnSubmissionFailureRetryInterval *int64 `json:"onSubmissionFailureRetryInterval,omitempty"`

	// OnFailureRetryInterval is the interval in seconds between retries on failed runs.
	// +optional
	OnFailureRetryInterval *int64 `json:"onFailureRetryInterval,omitempty"`
}

type RestartPolicyType string

const (
	RestartPolicyNever     RestartPolicyType = "Never"
	RestartPolicyOnFailure RestartPolicyType = "OnFailure"
	RestartPolicyAlways    RestartPolicyType = "Always"
)
```

| `type` の値 | 動作 |
| --- | --- |
| `Never` | 再実行しない。アプリケーションが失敗したら `FAILED` で停止する |
| `OnFailure` | アプリケーションが失敗した場合に再実行する |
| `Always` | 成功・失敗にかかわらず常に再実行する |

再実行の制御に関するフィールドを次の表にまとめる。

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `onSubmissionFailureRetries` | `*int32` | `spark-submit` の送信失敗時の再試行回数 |
| `onFailureRetries` | `*int32` | アプリケーション実行失敗時の再試行回数 |
| `onSubmissionFailureRetryInterval` | `*int64` | 送信失敗時の再試行間隔（秒） |
| `onFailureRetryInterval` | `*int64` | 実行失敗時の再試行間隔（秒） |

以下は自作の設定例である。

```yaml
spec:
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
    onFailureRetryInterval: 30
    onSubmissionFailureRetries: 2
    onSubmissionFailureRetryInterval: 10
```

この設定では、アプリケーションの実行失敗時に30秒間隔で最大3回再試行し、`spark-submit` の送信失敗時には10秒間隔で最大2回再試行する。

## 状態遷移と再実行の関係

第1章で示した状態機械において、再実行に関わる遷移を再確認する。

アプリケーションが `FAILED` または `SUBMISSION_FAILED` に遷移したとき、`restartPolicy` に再試行回数の余裕があれば `PENDING_RERUN` に遷移する。
`PENDING_RERUN` では `onFailureRetryInterval` または `onSubmissionFailureRetryInterval` で指定された間隔だけ待機し、その後 `SUBMITTED` に戻って再送信される。

`restartPolicy.type` が `Never` の場合、`FAILED`・`SUBMISSION_FAILED` は終端状態となり、再実行は発生しない。

## timeToLiveSeconds

**timeToLiveSeconds** はアプリケーションが終了（`COMPLETED` または `FAILED`）した後に、SparkApplication の Custom Resource を自動的に削除するまでの時間を指定する。

[api/v1beta2/sparkapplication_types.go の timeToLiveSeconds](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L129-L134)の定義は次のとおりである。

```go
	// TimeToLiveSeconds defines the Time-To-Live (TTL) duration in seconds for this SparkApplication
	// after its termination.
	// The SparkApplication object will be garbage collected if the current time is more than the
	// TimeToLiveSeconds since its termination.
	// +optional
	TimeToLiveSeconds *int64 `json:"timeToLiveSeconds,omitempty"`
```

TTL を設定しないと、完了した SparkApplication の Custom Resource はクラスタに残り続ける。
大量のバッチジョブを実行する環境では、TTL を設定して不要な Custom Resource を自動削除するのが望ましい。

## TTL の設定例

[examples/spark-pi-ttl.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-ttl.yaml#L16-L57)は `timeToLiveSeconds` を使った例である。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-ttl
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  sparkVersion: 4.0.1
  timeToLiveSeconds: 30
  driver:
    cores: 1
    memory: 512m
    serviceAccount: spark-operator-spark
  executor:
    instances: 1
    cores: 1
    memory: 512m
```

`timeToLiveSeconds: 30` により、アプリケーションが終了してから30秒後に SparkApplication の Custom Resource がガベージコレクションされる。

## 動作確認

TTL 付きのマニフェストを適用する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi-ttl.yaml
```

```text
sparkapplication.sparkoperator.k8s.io/spark-pi-ttl created
```

アプリケーションが完了したことを確認する。

```bash
kubectl get sparkapp spark-pi-ttl
```

```text
NAME         STATUS      ATTEMPTS   START              FINISH             AGE
spark-pi-ttl COMPLETED   1          2026-01-01 00:00   2026-01-01 00:01   1m
```

30秒後に Custom Resource が自動削除される。

```bash
kubectl get sparkapp spark-pi-ttl
```

```text
Error from server (NotFound): sparkapplications.sparkoperator.k8s.io "spark-pi-ttl" not found
```

## まとめ

- **RestartPolicy** は `type`（`Never`・`OnFailure`・`Always`）と再試行回数・間隔で、アプリケーションの自動再実行を制御する。
- `FAILED`・`SUBMISSION_FAILED` から `PENDING_RERUN` を経て `SUBMITTED` に戻る再実行ループは、`restartPolicy` の設定に基づいて発生する。
- **timeToLiveSeconds** はアプリケーション終了後の Custom Resource の自動削除までの時間を指定する。バッチジョブの大量実行環境では設定が望ましい。

## 関連する章

- [第1章 Spark Operator とは](../part00-introduction/01-what-is-spark-operator.md)（状態機械の全体像）
- [第4章 SparkApplication の基本構造](04-sparkapplication-spec.md)
- [第12章 ScheduledSparkApplication](../part03-other-crds/12-scheduledsparkapplication.md)（定期実行での再実行との違い）
