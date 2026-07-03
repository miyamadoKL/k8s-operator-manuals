# 第2章 インストール

> - [charts/spark-operator-chart/values.yaml L31-L43](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L31-L43)
> - [charts/spark-operator-chart/values.yaml L67-L98](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L67-L98)
> - [charts/spark-operator-chart/values.yaml L291-L322](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L291-L322)
> - [charts/spark-operator-chart/values.yaml L446-L475](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L446-L475)
> - [charts/spark-operator-chart/Chart.yaml L17-L38](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/Chart.yaml#L17-L38)

## この章でできるようになること

- Helm chart を使って Spark Operator をインストールできる。
- 主要な Helm values の設定項目を理解し、必要に応じてカスタマイズできる。
- インストールした Spark Operator をアンインストールできる。

## 前提

- Kubernetes クラスタ（バージョン1.16以上）が利用可能であること。
- `kubectl`（バージョン1.16以上）がインストール済みであること。
- `helm`（バージョン3以上）がインストール済みであること。

## Helm chart によるインストール

### リポジトリの追加

Spark Operator の Helm chart リポジトリを登録する。

```bash
helm repo add spark-operator https://kubeflow.github.io/spark-operator
helm repo update
```

### インストールコマンド

`spark-operator` Namespace にデフォルト設定でインストールする。

```bash
helm install spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --create-namespace \
    --wait
```

`--wait` を指定すると、Helm はすべての Pod が Ready になるまで待機する。

インストールが成功すると、次の出力が得られる。

```text
NAME: spark-operator
LAST DEPLOYED: ...
NAMESPACE: spark-operator
STATUS: deployed
REVISION: 1
```

### 動作確認

controller と webhook の Pod が Running になっていることを確認する。

```bash
kubectl get pods -n spark-operator
```

```text
NAME                                            READY   STATUS    RESTARTS   AGE
spark-operator-controller-xxxxx-yyyyy           1/1     Running   0          1m
spark-operator-webhook-xxxxx-yyyyy              1/1     Running   0          1m
```

## 主要な Helm values

[Chart.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/Chart.yaml#L17-L38)で、chart のバージョンは `2.5.1`、appVersion は `2.5.1` である。

[values.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml)から、導入時に確認すべき主要な設定項目を抽出する。

### イメージ設定

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

`tag` を空欄のままにすると、chart の `appVersion`（`2.5.1`）が使用される。
プライベートレジストリから取得する場合は `pullSecrets` に Secret 名を指定する。

### controller の設定

[values.yaml の controller セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L67-L98)の主要な項目は次のとおりである。

```yaml
controller:
  # -- Number of replicas of controller.
  replicas: 1

  # -- Feature gates to enable or disable specific features.
  featureGates:
  - name: PartialRestart
    enabled: false
  - name: LoadSparkDefaults
    enabled: false

  # -- The number of old history to retain to allow rollback.
  revisionHistoryLimit: 10

  leaderElection:
    # -- Specifies whether to enable leader election for controller.
    enable: true

  # -- Reconcile concurrency, higher values might increase memory usage.
  workers: 10

  # -- Configure the verbosity of logging, can be one of `debug`, `info`, `error`.
  logLevel: info

  # -- Configure the encoder of logging, can be one of `console` or `json`.
  logEncoder: console
```

| 項目 | デフォルト値 | 説明 |
| --- | --- | --- |
| `controller.replicas` | `1` | controller のレプリカ数 |
| `controller.workers` | `10` | リコンサイルの並列実行数。大きい値はメモリ使用量を増やす |
| `controller.logLevel` | `info` | ログの冗長度（`debug`・`info`・`error`） |
| `controller.leaderElection.enable` | `true` | レプリカ2台以上でのリーダ選出の可否 |
| `controller.featureGates` | すべて `false` | 実験的機能の有効化（`PartialRestart`・`LoadSparkDefaults`） |

### webhook の設定

[values.yaml の webhook セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L291-L322)の主要な項目は次のとおりである。

```yaml
webhook:
  # -- Specifies whether to enable webhook.
  enable: true

  # -- Number of replicas of webhook server.
  replicas: 1

  # -- Specifies webhook port.
  port: 9443

  # -- Specifies how unrecognized errors are handled.
  # Available options are `Ignore` or `Fail`.
  failurePolicy: Fail

  # -- Specifies the timeout seconds of the webhook, the value must be between 1 and 30.
  timeoutSeconds: 10
```

| 項目 | デフォルト値 | 説明 |
| --- | --- | --- |
| `webhook.enable` | `true` | Mutating Admission Webhook の可否 |
| `webhook.failurePolicy` | `Fail` | Webhook 呼び出し失敗時の挙動（`Ignore` で無視・`Fail` で拒否） |
| `webhook.timeoutSeconds` | `10` | Webhook のタイムアウト秒（1〜30） |

### Spark ジョブの設定

[values.yaml の spark セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L446-L475)は次のとおりである。

```yaml
spark:
  # -- List of namespaces where to run spark jobs.
  # If empty string is included, all namespaces will be allowed.
  # Namespaces specified here will be watched in addition to those matching jobNamespaceSelector.
  # Make sure the namespaces have already existed.
  jobNamespaces:
  - default

  serviceAccount:
    # -- Specifies whether to create a service account for spark applications.
    create: true

  rbac:
    # -- Specifies whether to create RBAC resources for spark applications.
    create: true
```

`spark.jobNamespaces` は Spark アプリケーションの実行を許可する Namespace の一覧である。
デフォルトは `default` のみである。
空文字列 `""` を含めると、すべての Namespace が許可される。

## カスタマイズ例

`spark.jobNamespaces` に複数の Namespace を指定し、ログレベルを `debug` に変更する例は次のとおりである。
以下は自作の設定例である。

```bash
helm install spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --create-namespace \
    --set spark.jobNamespaces={default,spark-jobs} \
    --set controller.logLevel=debug \
    --wait
```

## アンインストール

Helm release を削除すると、controller と webhook の Deployment が停止する。

```bash
helm uninstall spark-operator --namespace spark-operator
```

CRD は Helm で管理されないため、`helm uninstall` では削除されない。
CRD を併せて削除する場合は次のコマンドを実行する。

```bash
kubectl delete crd \
    sparkapplications.sparkoperator.k8s.io \
    scheduledsparkapplications.sparkoperator.k8s.io \
    sparkconnects.sparkoperator.k8s.io
```

CRD を削除すると、その配下に存在したすべての Custom Resource も削除される。
本番環境では注意すること。

Namespace ごと削除する場合は次のとおりである。

```bash
kubectl delete namespace spark-operator
```

## まとめ

- Spark Operator は Helm chart で `spark-operator` Namespace にインストールする。
- 主要な設定項目として、`image`（コンテナイメージ）、`controller`（リコンサイル並列度・ログレベル）、`webhook`（Webhook の可否とタイムアウト）、`spark`（ジョブ実行 Namespace）がある。
- アンインストール時は `helm uninstall` で release を削除し、必要に応じて CRD と Namespace を削除する。

## 関連する章

- [第1章 Spark Operator とは](01-what-is-spark-operator.md)
- [第3章 クイックスタート](03-quickstart.md)
