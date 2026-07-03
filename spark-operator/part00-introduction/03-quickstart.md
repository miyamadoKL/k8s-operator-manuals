# 第3章 クイックスタート

> - [examples/spark-pi.yaml L16-L62](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi.yaml#L16-L62)

## この章でできるようになること

- SparkApplication のマニフェストを Kubernetes クラスタに適用して Spark ジョブを実行できる。
- `kubectl get sparkapp` でジョブの状態を確認できる。
- 実行した SparkApplication を削除できる。

## 前提

- 第2章の手順に従って Spark Operator がインストール済みであること。
- `kubectl` でクラスタにアクセスできること。
- Spark Operator が `default` Namespace の SparkApplication を監視する設定であること（Helm values の `spark.jobNamespaces` に `default` が含まれている）。

## SparkApplication の適用

[examples/spark-pi.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi.yaml#L16-L62)は円周率を計算する最小構成の SparkApplication である。

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
  driver:
    labels:
      version: 4.0.0
    cores: 1
    memory: 512m
    serviceAccount: spark-operator-spark
  executor:
    labels:
      version: 4.0.0
    instances: 1
    cores: 1
    memory: 512m
```

このマニフェストの各フィールドは次の意味を持つ。

| フィールド | 値 | 説明 |
| --- | --- | --- |
| `spec.type` | `Scala` | アプリケーションの言語（`Java`・`Python`・`R` も指定可能） |
| `spec.mode` | `cluster` | デプロイモード（`cluster` では driver がクラスタ内で実行される） |
| `spec.image` | `docker.io/library/spark:4.0.1` | driver・executor で使用するコンテナイメージ |
| `spec.mainClass` | `org.apache.spark.examples.SparkPi` | Spark アプリケーションのエントリポイントとなるクラス |
| `spec.mainApplicationFile` | `local:///opt/spark/examples/jars/spark-examples.jar` | アプリケーションの JAR ファイルパス |
| `spec.arguments` | `["5000"]` | アプリケーションに渡す引数（サンプリング回数） |
| `spec.driver.cores` | `1` | driver Pod の CPU コア数 |
| `spec.driver.memory` | `512m` | driver Pod のメモリ量 |
| `spec.executor.instances` | `1` | executor Pod のインスタンス数 |
| `spec.executor.cores` | `1` | executor Pod の CPU コア数 |
| `spec.executor.memory` | `512m` | executor Pod のメモリ量 |

`kubectl apply` でこのマニフェストを適用する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi.yaml
```

```text
sparkapplication.sparkoperator.k8s.io/spark-pi created
```

## 状態の確認

`kubectl get sparkapp` で SparkApplication の状態を確認する。

```bash
kubectl get sparkapp spark-pi
```

起動直後は `SUBMITTED` 状態になる。

```text
NAME       AGE
spark-pi   10s
```

driver Pod と executor Pod が起動し、アプリケーションが実行されると `RUNNING` に遷移する。

```bash
kubectl get pods -n default
```

```text
NAME                                READY   STATUS    RESTARTS   AGE
spark-pi-driver                     1/1     Running   0          30s
spark-pi-xxxxx-exec-1               1/1     Running   0          25s
```

アプリケーションが完了すると、状態は `COMPLETED` になる。

```bash
kubectl get sparkapp spark-pi
```

```text
NAME       STATUS      ATTEMPTS   START              FINISH             AGE
spark-pi   COMPLETED   1          2026-01-01 00:00   2026-01-01 00:01   2m
```

driver Pod のログで、円周率の計算結果を確認できる。

```bash
kubectl logs spark-pi-driver -n default
```

```text
Pi is roughly 3.1415xxxx
```

## SparkApplication の削除

実行が完了した SparkApplication を削除する。

```bash
kubectl delete sparkapp spark-pi -n default
```

```text
sparkapplication.sparkoperator.k8s.io "spark-pi" deleted
```

削除すると、関連する driver Pod や executor Pod も自動的に削除される。

## トラブルシューティング

### `SUBMISSION_FAILED` 状態になる

`kubectl get sparkapp` で `Status` が `SUBMISSION_FAILED` になっている場合、`spark-submit` の実行に失敗している。

driver Pod のイベントを確認する。

```bash
kubectl describe sparkapp spark-pi -n default
```

よくある原因は次のとおりである。

- **ServiceAccount の不足**：`spec.driver.serviceAccount` に指定した ServiceAccount が存在しない。
  Helm のデフォルトでは `spark-operator-spark` が `default` Namespace に作成される。
- **イメージの取得失敗**：`spec.image` に指定したイメージが pull できない。
  `kubectl describe pod spark-pi-driver` で `ImagePullBackOff` のイベントが出ていないか確認する。

### Pod が `Pending` のまま起動しない

リソースが不足している可能性がある。

```bash
kubectl describe pod spark-pi-driver -n default
```

`Events` に `Insufficient cpu` や `Insufficient memory` がある場合、クラスタのノードリソースを確認する。

## まとめ

- `examples/spark-pi.yaml` を `kubectl apply` すると、Spark Operator が `spark-submit` を自動実行し、driver Pod と executor Pod を起動する。
- `kubectl get sparkapp` で状態遷移（`SUBMITTED` → `RUNNING` → `COMPLETED`）を追跡できる。
- 不要になった SparkApplication は `kubectl delete sparkapp` で削除する。

## 関連する章

- [第2章 インストール](02-installation.md)
- [第4章 SparkApplication の基本構造](../part01-sparkapplication-basics/04-sparkapplication-spec.md)
