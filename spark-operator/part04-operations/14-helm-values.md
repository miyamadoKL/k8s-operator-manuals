# 第14章 Helm values リファレンス

> - [charts/spark-operator-chart/values.yaml L31-L43](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L31-L43)
> - [charts/spark-operator-chart/values.yaml L67-L98](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L67-L98)
> - [charts/spark-operator-chart/values.yaml L144-L170](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L144-L170)
> - [charts/spark-operator-chart/values.yaml L234-L289](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L234-L289)
> - [charts/spark-operator-chart/values.yaml L291-L322](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L291-L322)
> - [charts/spark-operator-chart/values.yaml L446-L475](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L446-L475)
> - [charts/spark-operator-chart/values.yaml L477-L509](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L477-L509)
> - [charts/spark-operator-chart/values.yaml L511-L528](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L511-L528)

## この章でできるようになること

- Helm chart の `values.yaml` に含まれる主要な設定項目をセクション別に把握できる。
- 運用で変更すべき項目（リソース、並列度、ログレベル、名前空間制限等）を適切に設定できる。
- cert-manager 連携や Prometheus メトリクス公開の設定を理解できる。

## 前提

- 第2章で Helm chart によるインストールを経験していること。

## values.yaml の全体構成

[charts/spark-operator-chart/values.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml)は528行からなり、主に次のセクションで構成される。

| セクション | 行範囲 | 概要 |
| --- | --- | --- |
| `image` | 31-43 | コントローラのコンテナイメージ設定 |
| `hook` | 45-65 | CRD 更新用 Helm hook の設定 |
| `controller` | 67-289 | コントローラのDeployment設定 |
| `webhook` | 291-444 | Webhook サーバの設定 |
| `spark` | 446-475 | Spark アプリケーションの実行設定 |
| `prometheus` | 477-509 | コントローラのメトリクス設定 |
| `certManager` | 511-528 | cert-manager 連携の設定 |

本章では、全項目の逐語列挙ではなく、運用で触る項目を重点的に解説する。

## image セクション

[values.yaml の image セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L31-L43)は次のとおりである。

```yaml
image:
  # -- Image registry.
  registry: ghcr.io
  # -- Image repository.
  repository: kubeflow/spark-operator/controller
  # -- Image tag.
  # @default -- If not set, the chart appVersion will be used.
  tag: ""
  # -- Image pull policy.
  pullPolicy: IfNotPresent
  # -- Image pull secrets for private image registry.
  pullSecrets: []
```

| 項目 | デフォルト | 説明 |
| --- | --- | --- |
| `registry` | `ghcr.io` | イメージのレジストリ |
| `repository` | `kubeflow/spark-operator/controller` | イメージのリポジトリ |
| `tag` | `""`（appVersion を使用） | イメージのタグ |
| `pullPolicy` | `IfNotPresent` | イメージの取得ポリシー |
| `pullSecrets` | `[]` | プライベートレジストリの認証 Secret |

プライベートレジストリを使用する場合は `registry`・`repository`・`tag` を変更し、`pullSecrets` に Secret 名を指定する。

## controller セクション

### 基本設定

[values.yaml の controller 基本設定](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L67-L98)は次のとおりである。

```yaml
controller:
  # -- Number of replicas of controller.
  replicas: 1

  # -- Reconcile concurrency, higher values might increase memory usage.
  workers: 10

  # -- Configure the verbosity of logging, can be one of `debug`, `info`, `error`.
  logLevel: info

  # -- Configure the encoder of logging, can be one of `console` or `json`.
  logEncoder: console
```

| 項目 | デフォルト | 説明 |
| --- | --- | --- |
| `replicas` | `1` | コントローラのレプリカ数 |
| `workers` | `10` | リコンサイルの並列実行数 |
| `logLevel` | `info` | ログの冗長度（`debug`・`info`・`error`） |
| `logEncoder` | `console` | ログのフォーマット（`console`・`json`） |

`workers` を増やすとリコンサイルの並列度が上がるが、メモリ使用量も増加する。
大量の SparkApplication を処理する環境では `workers` を増やし、`resources.limits.memory` も合わせて増やす必要がある。

### リーダ選出

[values.yaml の leaderElection](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L81-L89)は次のとおりである。

```yaml
  leaderElection:
    # -- Specifies whether to enable leader election for controller.
    enable: true
    # -- Leader election lease duration.
    leaseDuration: 15s
    # -- Leader election renew deadline.
    renewDeadline: 10s
    # -- Leader election retry period.
    retryPeriod: 2s
```

`replicas: 2` 以上で高可用性を構成する場合、`leaderElection.enable: true`（デフォルト）でリーダ選出が有効になる。
`leaseDuration`・`renewDeadline`・`retryPeriod` は通常の変更要項ではない。

### バッチスケジューラ

[values.yaml の batchScheduler](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L144-L154)は次のとおりである。

```yaml
  batchScheduler:
    # -- Specifies whether to enable batch scheduler for spark jobs scheduling.
    enable: false
    # -- Specifies a list of kube-scheduler names for scheduling Spark pods.
    kubeSchedulerNames: []
    # -- Default batch scheduler to be used if not specified by the user.
    default: ""
```

Volcano・YuniKorn を使用する場合は `enable: true` を設定する（第9章参照）。

### ServiceAccount と RBAC

[values.yaml の controller.serviceAccount と rbac](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L156-L170)は次のとおりである。

```yaml
  serviceAccount:
    create: true
    name: ""
    annotations: {}
    automountServiceAccountToken: true

  rbac:
    create: true
    annotations: {}
```

`serviceAccount.create` と `rbac.create` はデフォルトで `true` である。
既存の ServiceAccount を使用する場合は `create: false` にし、`name` で既存の ServiceAccount 名を指定する。

### リソース制限

[values.yaml の resources](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L234-L238)は次のとおりである。

```yaml
  # -- Pod resource requests and limits for controller containers.
  # Note, that each job submission will spawn a JVM within the controller pods using "/usr/local/openjdk-11/bin/java -Xmx128m".
  # Kubernetes may kill these Java processes at will to enforce resource limits. When that happens, you will see the following error:
  # 'failed to run spark-submit for SparkApplication [...]: signal: killed' - when this happens, you may want to increase memory limits.
  resources: {}
```

デフォルトではリソース制限が設定されていない。
本番環境では必ず `resources.limits.memory` を設定する。
`signal: killed` エラーが発生した場合は、メモリ制限を増やす必要がある。

### Workqueue Rate Limiter

[values.yaml の workqueueRateLimiter](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L278-L289)は次のとおりである。

```yaml
  workqueueRateLimiter:
    # -- Specifies the average rate of items process by the workqueue rate limiter.
    bucketQPS: 50
    # -- Specifies the maximum number of items that can be in the workqueue at any given time.
    bucketSize: 500
    maxDelay:
      enable: true
      duration: 6h
```

`bucketQPS` と `bucketSize` はリコンサイルキューのレート制限を制御する。
大量のイベントを処理する環境で `bucketSize` を増やすと、イベントの喪失を防げる。

## webhook セクション

[values.yaml の webhook セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L291-L322)の主要な項目は次のとおりである。

```yaml
webhook:
  enable: true
  replicas: 1
  logLevel: info
  logEncoder: console
  port: 9443
  failurePolicy: Fail
  timeoutSeconds: 10
```

| 項目 | デフォルト | 説明 |
| --- | --- | --- |
| `enable` | `true` | Webhook の可否 |
| `replicas` | `1` | Webhook サーバのレプリカ数 |
| `failurePolicy` | `Fail` | Webhook 呼び出し失敗時の挙動（`Ignore`・`Fail`） |
| `timeoutSeconds` | `10` | Webhook のタイムアウト秒（1〜30） |

`failurePolicy: Fail` は Webhook が応答しない場合に Pod の作成を拒否する。
Webhook の一時的な障害で SparkApplication の作成が失敗する場合は `Ignore` に変更できるが、デフォルト値の注入が行われなくなる。

## spark セクション

[values.yaml の spark セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L446-L475)は次のとおりである。

```yaml
spark:
  jobNamespaces:
  - default
  jobNamespaceSelector: ""
  serviceAccount:
    create: true
    name: ""
    annotations: {}
    automountServiceAccountToken: true
  rbac:
    create: true
    annotations: {}
```

| 項目 | デフォルト | 説明 |
| --- | --- | --- |
| `jobNamespaces` | `[default]` | Spark アプリケーションの実行を許可する Namespace の一覧 |
| `jobNamespaceSelector` | `""` | Namespace をラベルセレクタでフィルタする式 |
| `serviceAccount.create` | `true` | Spark アプリケーション用の ServiceAccount を作成するかどうか |
| `rbac.create` | `true` | Spark アプリケーション用の RBAC を作成するかどうか |

`jobNamespaces` に `""`（空文字列）を含めると、すべての Namespace が許可される。
特定の Namespace だけを許可する場合は、許可する Namespace 名を列挙する。

## prometheus セクション

[values.yaml の prometheus セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L477-L509)は次のとおりである。

```yaml
prometheus:
  metrics:
    enable: true
    port: 8080
    portName: metrics
    endpoint: /metrics
    prefix: ""
    jobSubmitLatencyBuckets: "0.5,1,2,4,8,16,32,64,128,256"
    jobStartLatencyBuckets: "30,60,90,120,150,180,210,240,270,300"
    labels: ""
  podMonitor:
    create: false
    labels: {}
    jobLabel: spark-operator-podmonitor
    podMetricsEndpoint:
      scheme: http
      interval: 5s
```

| 項目 | デフォルト | 説明 |
| --- | --- | --- |
| `metrics.enable` | `true` | コントローラのメトリクス公開の可否 |
| `metrics.port` | `8080` | メトリクスエンドポイントのポート |
| `metrics.endpoint` | `/metrics` | メトリクスエンドポイントのパス |
| `podMonitor.create` | `false` | Prometheus Operator 用の PodMonitor を作成するかどうか |

Prometheus Operator を使用している場合は `podMonitor.create: true` を設定すると、PodMonitor が自動作成される。

## certManager セクション

[values.yaml の certManager セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L511-L528)は次のとおりである。

```yaml
certManager:
  enable: false
  issuerRef: {}
  duration: ""
  renewBefore: ""
```

Webhook の TLS 証明書に cert-manager を使用する場合は `enable: true` を設定する。
デフォルトでは Webhook サーバが自己署名証明書を自動生成する。
cert-manager を使用する場合は `issuerRef` で Issuer を指定する。

## まとめ

- `values.yaml` は `image`・`controller`・`webhook`・`spark`・`prometheus`・`certManager` のセクションで構成される。
- 運用で変更すべき主要項目は `controller.workers`（並列度）、`controller.resources`（リソース制限）、`spark.jobNamespaces`（実行許可 Namespace）、`webhook.failurePolicy`（Webhook 失敗時の挙動）である。
- `signal: killed` エラーが発生した場合は `controller.resources.limits.memory` を増やす。

## 関連する章

- [第2章 インストール](../part00-introduction/02-installation.md)
- [第16章 運用とトラブルシューティング](16-operations-troubleshooting.md)
