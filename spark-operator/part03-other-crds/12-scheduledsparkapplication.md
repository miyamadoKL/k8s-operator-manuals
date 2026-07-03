# 第12章 ScheduledSparkApplication

> - [api/v1beta2/scheduledsparkapplication_types.go L26-L55](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/scheduledsparkapplication_types.go#L26-L55)
> - [api/v1beta2/scheduledsparkapplication_types.go L57-L78](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/scheduledsparkapplication_types.go#L57-L78)
> - [api/v1beta2/scheduledsparkapplication_types.go L93-L100](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/scheduledsparkapplication_types.go#L93-L100)
> - [api/v1beta2/scheduledsparkapplication_types.go L112-L131](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/scheduledsparkapplication_types.go#L112-L131)
> - [config/crd/bases/sparkoperator.k8s.io_scheduledsparkapplications.yaml L1-L17](https://github.com/kubeflow/spark-operator/blob/v2.5.1/config/crd/bases/sparkoperator.k8s.io_scheduledsparkapplications.yaml#L1-L17)
> - [examples/spark-pi-scheduled.yaml L17-L62](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-scheduled.yaml#L17-L62)

## この章でできるようになること

- **ScheduledSparkApplication** の spec 構造を理解し、定期実行の Spark ジョブを設定できる。
- Cron 形式のスケジュール、並行実行ポリシー、実行履歴の保持数を制御できる。
- 第7章の RestartPolicy による再実行との違いを説明できる。

## 前提

- 第4章で SparkApplication の基本構造を理解していること。
- 第7章で RestartPolicy による再実行の仕組みを理解していること。

## ScheduledSparkApplication の概要

**ScheduledSparkApplication** は Cron 形式のスケジュールで SparkApplication を自動生成する Custom Resource である。
Kubernetes の CronJob と同様に、定期的な Spark ジョブの実行を宣言的に管理できる。

[config/crd/bases/sparkoperator.k8s.io_scheduledsparkapplications.yaml の CRD 定義](https://github.com/kubeflow/spark-operator/blob/v2.5.1/config/crd/bases/sparkoperator.k8s.io_scheduledsparkapplications.yaml#L1-L17)は次のとおりである。

```yaml
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    api-approved.kubernetes.io: https://github.com/kubeflow/spark-operator/pull/1298
    controller-gen.kubebuilder.io/version: v0.17.1
  name: scheduledsparkapplications.sparkoperator.k8s.io
spec:
  group: sparkoperator.k8s.io
  names:
    kind: ScheduledSparkApplication
    listKind: ScheduledSparkApplicationList
    plural: scheduledsparkapplications
    shortNames:
    - scheduledsparkapp
    singular: scheduledsparkapplication
```

ショートネームは `scheduledsparkapp` であり、`kubectl get scheduledsparkapp` で状態を確認できる。

## ScheduledSparkApplicationSpec

[api/v1beta2/scheduledsparkapplication_types.go の ScheduledSparkApplicationSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/scheduledsparkapplication_types.go#L26-L55)は次のように定義されている。

```go
// ScheduledSparkApplicationSpec defines the desired state of ScheduledSparkApplication.
type ScheduledSparkApplicationSpec struct {
	// Schedule is a cron schedule on which the application should run.
	Schedule string `json:"schedule"`
	// TimeZone is the time zone in which the cron schedule will be interpreted in.
	// ...
	// Defaults to "Local".
	TimeZone string `json:"timeZone,omitempty"`
	// Template is a template from which SparkApplication instances can be created.
	Template SparkApplicationSpec `json:"template"`
	// Suspend is a flag telling the controller to suspend subsequent runs of the application if set to true.
	// ...
	// Defaults to false.
	Suspend *bool `json:"suspend,omitempty"`
	// ConcurrencyPolicy is the policy governing concurrent SparkApplication runs.
	ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`
	// SuccessfulRunHistoryLimit is the number of past successful runs of the application to keep.
	// ...
	// Defaults to 1.
	SuccessfulRunHistoryLimit *int32 `json:"successfulRunHistoryLimit,omitempty"`
	// FailedRunHistoryLimit is the number of past failed runs of the application to keep.
	// ...
	// Defaults to 1.
	FailedRunHistoryLimit *int32 `json:"failedRunHistoryLimit,omitempty"`
}
```

各フィールドの役割を次の表にまとめる。

| フィールド | 型 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `schedule` | `string` | （必須） | Cron 形式のスケジュール（`0 2 * * *` 等）または `@every 3m` 形式 |
| `timeZone` | `string` | `Local` | スケジュールを解釈するタイムゾーン（IANA 形式、例: `Asia/Tokyo`） |
| `template` | `SparkApplicationSpec` | （必須） | SparkApplication のテンプレート |
| `suspend` | `*bool` | `false` | 以降の実行を停止するかどうか |
| `concurrencyPolicy` | `ConcurrencyPolicy` | （なし） | 並行実行のポリシー（`Allow`・`Forbid`・`Replace`） |
| `successfulRunHistoryLimit` | `*int32` | `1` | 保持する成功実行の履歴数 |
| `failedRunHistoryLimit` | `*int32` | `1` | 保持する失敗実行の履歴数 |

## スケジュールの指定

`schedule` フィールドには Cron 式または `@every` 形式を指定する。

| 形式 | 例 | 説明 |
| --- | --- | --- |
| Cron 式 | `0 2 * * *` | 毎日午前2時に実行 |
| Cron 式 | `*/15 * * * *` | 15分ごとに実行 |
| `@every` | `@every 3m` | 3分ごとに実行 |
| `@every` | `@every 1h` | 1時間ごとに実行 |

`timeZone` を指定すると、スケジュールを特定のタイムゾーンで解釈できる。
省略すると `Local`（コントローラのローカルタイムゾーン）が使用される。

## 並行実行ポリシー

**ConcurrencyPolicy** は、前回の実行が完了していない場合に次の実行をどう扱うかを制御する。

[api/v1beta2/scheduledsparkapplication_types.go の ConcurrencyPolicy](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/scheduledsparkapplication_types.go#L112-L122)は次のように定義されている。

```go
type ConcurrencyPolicy string

const (
	// ConcurrencyAllow allows SparkApplications to run concurrently.
	ConcurrencyAllow ConcurrencyPolicy = "Allow"
	// ConcurrencyForbid forbids concurrent runs of SparkApplications, skipping the next run if the previous
	// one hasn't finished yet.
	ConcurrencyForbid ConcurrencyPolicy = "Forbid"
	// ConcurrencyReplace kills the currently running SparkApplication instance and replaces it with a new one.
	ConcurrencyReplace ConcurrencyPolicy = "Replace"
)
```

各ポリシーの動作を次の表にまとめる。

| ポリシー | 動作 |
| --- | --- |
| `Allow` | 前回の実行が完了していなくても、新しい実行を開始する |
| `Forbid` | 前回の実行が完了していない場合、新しい実行をスキップする |
| `Replace` | 前回の実行を強制終了し、新しい実行を開始する |

## ScheduledSparkApplicationStatus

[api/v1beta2/scheduledsparkapplication_types.go の ScheduledSparkApplicationStatus](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/scheduledsparkapplication_types.go#L57-L78)は次のように定義されている。

```go
// ScheduledSparkApplicationStatus defines the observed state of ScheduledSparkApplication.
type ScheduledSparkApplicationStatus struct {
	// LastRun is the time when the last run of the application started.
	// +nullable
	LastRun metav1.Time `json:"lastRun,omitempty"`
	// NextRun is the time when the next run of the application will start.
	// +nullable
	NextRun metav1.Time `json:"nextRun,omitempty"`
	// LastRunName is the name of the SparkApplication for the most recent run of the application.
	LastRunName string `json:"lastRunName,omitempty"`
	// PastSuccessfulRunNames keeps the names of SparkApplications for past successful runs.
	PastSuccessfulRunNames []string `json:"pastSuccessfulRunNames,omitempty"`
	// PastFailedRunNames keeps the names of SparkApplications for past failed runs.
	PastFailedRunNames []string `json:"pastFailedRunNames,omitempty"`
	// ScheduleState is the current scheduling state of the application.
	ScheduleState ScheduleState `json:"scheduleState,omitempty"`
	// Reason tells why the ScheduledSparkApplication is in the particular ScheduleState.
	Reason string `json:"reason,omitempty"`
}
```

`status.lastRun` と `status.nextRun` で、最後の実行時刻と次回の実行時刻を確認できる。
`status.pastSuccessfulRunNames` と `status.pastFailedRunNames` で、保持されている実行履歴の SparkApplication 名を確認できる。

## 実行例

[examples/spark-pi-scheduled.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-scheduled.yaml#L17-L62)は次のとおりである。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: ScheduledSparkApplication
metadata:
  name: spark-pi-scheduled
  namespace: default
spec:
  schedule: "@every 3m"
  concurrencyPolicy: Allow
  template:
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
      cores: 1
      memory: 512m
      serviceAccount: spark-operator-spark
    executor:
      instances: 1
      cores: 1
      memory: 512m
```

この設定では、3分ごとに SparkApplication が生成される。
`concurrencyPolicy: Allow` により、前回の実行が完了していなくても新しい実行が開始される。
`template.restartPolicy.type: Never` により、生成された SparkApplication は失敗しても再実行されない。

## 動作確認

ScheduledSparkApplication を適用する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi-scheduled.yaml
```

```text
scheduledsparkapplication.sparkoperator.k8s.io/spark-pi-scheduled created
```

状態を確認する。

```bash
kubectl get scheduledsparkapp spark-pi-scheduled
```

```text
NAME                 SCHEDULE    TIMEZONE   SUSPEND   LAST RUN   LAST RUN NAME   AGE
spark-pi-scheduled   @every 3m                         <none>                     10s
```

3分後に最初の SparkApplication が生成される。

```bash
kubectl get sparkapp
```

```text
NAME                            STATUS      ATTEMPTS   START              FINISH             AGE
spark-pi-scheduled-xxxxx        SUBMITTED   0          2026-01-01 00:03   <none>             5s
```

## RestartPolicy との違い

第7章で説明した **RestartPolicy** と **ScheduledSparkApplication** は、どちらも Spark アプリケーションの再実行に関わるが、その目的と仕組みは異なる。

| 項目 | RestartPolicy | ScheduledSparkApplication |
| --- | --- | --- |
| 目的 | 失敗したアプリケーションの自動再実行 | 定期的なスケジュールでの実行 |
| トリガー | アプリケーションの失敗（`FAILED`・`SUBMISSION_FAILED`） | Cron スケジュール |
| 生成されるリソース | 同じ SparkApplication を再送信 | 新しい SparkApplication を生成 |
| 履歴管理 | `submissionAttempts`・`executionAttempts` のみ | `successfulRunHistoryLimit`・`failedRunHistoryLimit` で保持数を制御 |
| 並行実行の制御 | なし | `concurrencyPolicy` で制御 |

**RestartPolicy** は、同じ SparkApplication が失敗した場合に再試行する仕組みである。
`onFailureRetries` で再試行回数を指定し、`onFailureRetryInterval` で再試行間隔を制御する。
再実行は同じ Custom Resource で行われるため、履歴は `submissionAttempts`・`executionAttempts` のカウンタとして記録される。

**ScheduledSparkApplication** は、スケジュールに従って新しい SparkApplication を生成する仕組みである。
各実行は独立した SparkApplication として作成され、`successfulRunHistoryLimit`・`failedRunHistoryLimit` で保持する履歴数を制御できる。
古い実行は自動的に削除される。

## まとめ

- **ScheduledSparkApplication** は Cron 形式のスケジュールで SparkApplication を自動生成する。
- `concurrencyPolicy` で並行実行のポリシー（`Allow`・`Forbid`・`Replace`）を制御できる。
- `successfulRunHistoryLimit`・`failedRunHistoryLimit` で実行履歴の保持数を制御できる。
- **RestartPolicy** は失敗時の再試行、**ScheduledSparkApplication** はスケジュールに基づく定期実行である。前者は同じ Custom Resource を再送信し、後者は新しい Custom Resource を生成する。

## 関連する章

- [第7章 ライフサイクルと再実行](../part01-sparkapplication-basics/07-lifecycle.md)
- [第4章 SparkApplication の基本構造](../part01-sparkapplication-basics/04-sparkapplication-spec.md)
