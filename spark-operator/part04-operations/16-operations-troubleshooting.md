# 第16章 運用とトラブルシューティング

> - [charts/spark-operator-chart/values.yaml L67-L98](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L67-L98)
> - [charts/spark-operator-chart/values.yaml L234-L244](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L234-L244)
> - [charts/spark-operator-chart/values.yaml L45-L65](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L45-L65)
> - [docs/kustomize-installation.md L1-L48](https://github.com/kubeflow/spark-operator/blob/v2.5.1/docs/kustomize-installation.md#L1-L48)

## この章でできるようになること

- Spark Operator のアップグレード手順（Helm・CRD 更新を含む）を理解できる。
- コントローラのログを確認して問題を切り分けできる。
- よくある障害の原因と対処法を把握できる。

## 前提

- 第2章で Helm chart によるインストールを経験していること。
- 第14章で Helm values の主要な設定項目を把握していること。

## アップグレード

### Helm chart のアップグレード

Helm chart をアップグレードする場合は `helm upgrade` コマンドを使用する。

```bash
helm repo update
helm upgrade spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --reuse-values
```

`--reuse-values` を指定すると、既存の values を維持したままアップグレードされる。
values をリセットする場合は `--reuse-values` を省略し、`-f values.yaml` で新しい values ファイルを指定する。

### CRD の更新

Helm chart のアップグレードでは CRD は自動更新されない。
CRD を手動で更新する場合は次のコマンドを実行する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/config/crd/bases/sparkoperator.k8s.io_sparkapplications.yaml
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/config/crd/bases/sparkoperator.k8s.io_scheduledsparkapplications.yaml
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/config/crd/bases/sparkoperator.k8s.io_sparkconnects.yaml
```

[values.yaml の hook セクション](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L46-L48)で `hook.upgradeCrd: true` を設定すると、Helm の pre-upgrade hook で CRD が自動更新される。

```yaml
hook:
  # -- Whether to create a Helm pre-install/pre-upgrade hook Job to update CRDs.
  upgradeCrd: false
```

CRD の更新時に既存の Custom Resource が削除されることはない。
ただし、フィールドの削除や型の変更がある場合は、既存の Custom Resource が新しい CRD と互換性を持たない可能性がある。

## ログの確認

### コントローラのログ

コントローラのログを確認する。

```bash
kubectl logs -n spark-operator deployment/spark-operator-controller
```

ログレベルを `debug` に変更すると、詳細なリコンサイルのログが出力される。

```bash
helm upgrade spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --set controller.logLevel=debug
```

### Webhook のログ

Webhook サーバのログを確認する。

```bash
kubectl logs -n spark-operator deployment/spark-operator-webhook
```

Webhook のログで `Mutating SparkApplication` や `Mutating Pod` のエントリを確認すると、デフォルト注入の動作を追跡できる（第15章参照）。

## 運用パラメーター

### workers（リコンサイル並列度）

[values.yaml の controller.workers](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L91-L92)はリコンサイルの並列実行数を制御する。

```yaml
  # -- Reconcile concurrency, higher values might increase memory usage.
  workers: 10
```

`workers` を増やすとリコンサイルのスループットが上がるが、メモリ使用量も増加する。
大量の SparkApplication を処理する環境では `workers` を増やし、`resources.limits.memory` も合わせて増やす。

### leaderElection（リーダ選出）

[values.yaml の controller.leaderElection](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L81-L89)は高可用性構成でのリーダ選出を制御する。

```yaml
controller:
  leaderElection:
    enable: true
    leaseDuration: 15s
    renewDeadline: 10s
    retryPeriod: 2s
```

`replicas: 2` 以上で高可用性を構成する場合、`leaderElection.enable: true`（デフォルト）でリーダ選出が有効になる。
リーダ選出の詳細は Kubernetes の Leader Election のドキュメントを参照すること。

### resources（リソース制限）

[values.yaml の controller.resources](https://github.com/kubeflow/spark-operator/blob/v2.5.1/charts/spark-operator-chart/values.yaml#L234-L244)はコントローラ Pod のリソース制限を制御する。

```yaml
controller:
  resources: {}
    # limits:
    #   cpu: 100m
    #   memory: 300Mi
    # requests:
    #   cpu: 100m
    #   memory: 300Mi
```

本番環境では必ず `resources.limits.memory` を設定する。
デフォルトではリソース制限が設定されていないため、OOM Kill のリスクがある。

## よくある障害と切り分け

### signal: killed エラー

**症状**：SparkApplication の状態が `SUBMISSION_FAILED` になり、コントローラのログに `signal: killed` が出力される。

**原因**：`spark-submit` の実行時に JVM が起動するが、コントローラ Pod のメモリ制限を超えて Kubernetes に kill される。

**対処**：`controller.resources.limits.memory` を増やす。

```bash
helm upgrade spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --set controller.resources.limits.memory=1Gi
```

### Webhook timeout エラー

**症状**：SparkApplication の作成時に `context deadline exceeded` や `webhook timeout` エラーが発生する。

**原因**：Webhook サーバの応答が遅い、または Webhook サーバが応答していない。

**対処**：

1. Webhook サーバのログとリソース使用量を確認する。
2. `webhook.timeoutSeconds` を増やす（最大30秒）。
3. Webhook サーバのレプリカ数を増やす。

```bash
helm upgrade spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --set webhook.timeoutSeconds=20 \
    --set webhook.replicas=2
```

### ImagePullBackOff エラー

**症状**：driver Pod または executor Pod が `ImagePullBackOff` 状態で起動しない。

**原因**：`spec.image` で指定されたコンテナイメージが pull できない。

**対処**：

1. イメージ名とタグが正しいことを確認する。
2. プライベートレジストリの場合は `imagePullSecrets` が正しく設定されていることを確認する。

```bash
kubectl describe pod spark-pi-driver -n default
```

```text
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Warning  Failed     1m    kubelet            Failed to pull image "...": rpc error: code = Unknown desc = Error response from daemon: manifest for ... not found
```

### Pod が Pending のまま起動しない

**症状**：driver Pod または executor Pod が `Pending` 状態のままスケジューリングされない。

**原因**：クラスタに十分なリソースがない、または Node Selector・Affinity の条件を満たすノードがない。

**対処**：

1. `kubectl describe pod` でイベントを確認する。
2. `Insufficient cpu` や `Insufficient memory` の場合は、クラスタのノードリソースを確認する。
3. Node Selector・Affinity の条件を確認し、条件を満たすノードが存在することを確認する。

```bash
kubectl describe pod spark-pi-driver -n default
```

```text
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  1m    default-scheduler  0/3 nodes are available: 3 Insufficient memory.
```

### SparkApplication が COMPLETED にならない

**症状**：driver Pod が正常に終了しているにもかかわらず、SparkApplication の状態が `RUNNING` のまま変わらない。

**原因**：コントローラが driver Pod の状態変化を正しく検出できていない。

**対処**：

1. コントローラのログでリコンサイルのエラーを確認する。
2. `controller.driverPodCreationGracePeriod` を増やす（デフォルト `10s`）。

```bash
helm upgrade spark-operator spark-operator/spark-operator \
    --namespace spark-operator \
    --set controller.driverPodCreationGracePeriod=30s
```

## まとめ

- アップグレード時は `helm upgrade` でコントローラを更新し、必要に応じて CRD を手動で更新する。
- `controller.logLevel=debug` で詳細なログを出力できる。
- `signal: killed` エラーは `controller.resources.limits.memory` を増やすことで対処する。
- Webhook timeout エラーは `webhook.timeoutSeconds` を増やすか、Webhook サーバのリソースを確認することで対処する。
- ImagePullBackOff エラーはイメージ名と `imagePullSecrets` を確認することで対処する。
- Pod が Pending のまま起動しない場合は、クラスタのリソースと Node Selector・Affinity の条件を確認する。

## 関連する章

- [第2章 インストール](../part00-introduction/02-installation.md)
- [第7章 ライフサイクルと再実行](../part01-sparkapplication-basics/07-lifecycle.md)
- [第14章 Helm values リファレンス](14-helm-values.md)
- [第15章 Webhook とデフォルト注入](15-webhook.md)
