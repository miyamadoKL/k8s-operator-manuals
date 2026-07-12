# 第21章 Helm values リファレンス

> 本章で参照する公式リソース
>
> - [helm-charts/helm3/strimzi-kafka-operator/values.yaml L3-L36](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/values.yaml#L3-L36)
> - [helm-charts/helm3/strimzi-kafka-operator/values.yaml L117-L195](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/values.yaml#L117-L195)
> - [helm-charts/helm3/strimzi-kafka-operator/values.yaml L203-L214](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/values.yaml#L203-L214)
> - [install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml L41-L88](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml#L41-L88)

## この章でできるようになること

- Helm chart の主要 values を把握し、運用に合わせて上書きできる。
- `watchNamespaces` と `watchAnyNamespace` の違いを説明できる。
- `featureGates` と `fullReconciliationIntervalMs` の意味を理解できる。
- `helm get values` で反映結果を確認できる。

## 前提

[第2章 インストール](../part00-introduction/02-installation.md)で Helm による Cluster Operator インストールを理解していること。
`strimzi` Namespace に `strimzi-kafka-operator` という名前の Helm release が存在すること（未導入の場合は [第2章](../part00-introduction/02-installation.md)の手順 B を先に実行する）。

## Operator 本体の values

[helm-charts/helm3/strimzi-kafka-operator/values.yaml L3-L36](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/values.yaml#L3-L36)は次のとおりである。

```yaml
# Default replicas for the cluster operator
replicas: 1

# Contains `.Release.Namespace` by default
watchNamespaces: []
watchAnyNamespace: false

defaultImageRegistry: quay.io
defaultImageRepository: strimzi
defaultImageTag: 1.1.0

image:
  registry: ""
  repository: ""
  name: operator
  tag: ""
  # imagePullSecrets:
  #   - name: secretname
logVolume: co-config-volume
logConfigMap: strimzi-cluster-operator
logConfiguration: ""
logLevel: ${env:STRIMZI_LOG_LEVEL:-INFO}
fullReconciliationIntervalMs: 120000
operationTimeoutMs: 300000
kubernetesServiceDnsDomain: cluster.local
featureGates: ""
tmpDirSizeLimit: 1Mi

# Example on how to configure extraEnvs
# extraEnvs:
#   - name: JAVA_OPTS
#     value: "-Xms256m -Xmx256m"

extraEnvs: []
```

| キー | 説明 | デフォルト |
|---|---|---|
| `replicas` | Cluster Operator Deployment のレプリカ数 | `1` |
| `watchNamespaces` | 監視対象 Namespace のリスト（空なら Release Namespace のみ） | `[]` |
| `watchAnyNamespace` | 全 Namespace を監視するか | `false` |
| `defaultImageRegistry` | オペランドイメージのレジストリ | `quay.io` |
| `defaultImageTag` | オペランドイメージのタグ | `1.1.0` |
| `logLevel` | Operator のログレベル | `INFO` |
| `fullReconciliationIntervalMs` | 定期リコンサイル間隔（ミリ秒） | `120000` |
| `operationTimeoutMs` | 操作タイムアウト（ミリ秒） | `300000` |
| `featureGates` | 機能ゲート（`+` で有効化、`-` で無効化） | `""`（空） |
| `extraEnvs` | Deployment に追加する環境変数 | `[]` |

`defaultImageTag` は多くのオペランドに共通だが、Kafka Bridge は `kafkaBridge.image.tag: 1.0.0` を個別に持つ。

## リソースとオペランドイメージ

[helm-charts/helm3/strimzi-kafka-operator/values.yaml L117-L195](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/values.yaml#L117-L195)は次のとおりである。

```yaml
kafka:
  image:
    registry: ""
    repository: ""
    name: kafka
    tagPrefix: ""
kafkaConnect:
  image:
    registry: ""
    repository: ""
    name: kafka
    tagPrefix: ""
topicOperator:
  image:
    registry: ""
    repository: ""
    name: operator
    tag: ""
userOperator:
  image:
    registry:
    repository:
    name: operator
    tag: ""
kafkaInit:
  image:
    registry: ""
    repository: ""
    name: operator
    tag: ""
kafkaBridge:
  image:
    registry: ""
    repository:
    name: kafka-bridge
    tag: 1.0.0
kafkaExporter:
  image:
    registry: ""
    repository: ""
    name: kafka
    tagPrefix: ""
kafkaMirrorMaker2:
  image:
    registry: ""
    repository: ""
    name: kafka
    tagPrefix: ""
cruiseControl:
  image:
    registry: ""
    repository: ""
    name: kafka
    tagPrefix: ""
kanikoExecutor:
  image:
    registry: ""
    repository: ""
    name: kaniko-executor
    tag: ""
buildah:
  image:
    registry: ""
    repository: ""
    name: buildah
    tag: ""
mavenBuilder:
  image:
    registry: ""
    repository: ""
    name: maven-builder
    tag: ""
resources:
  limits:
    memory: 384Mi
    cpu: 1000m
  requests:
    memory: 384Mi
    cpu: 200m
```

各オペランド（`kafka`、`kafkaConnect`、`cruiseControl` 等）のイメージを個別に上書きできる。
Operator 自身の CPU とメモリは `resources` で制御する。

末尾の運用フラグは [helm-charts/helm3/strimzi-kafka-operator/values.yaml L203-L214](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/values.yaml#L203-L214)にある。

```yaml
createGlobalResources: true
# Create clusterroles that extend existing clusterroles to interact with strimzi crds
# Ref: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles
createAggregateRoles: false
# Override the exclude pattern for exclude some labels
labelsExclusionPattern: ""
# Controls whether Strimzi generates network policy resources (By default true)
generateNetworkPolicy: true
# Override the value for Connect build timeout
connectBuildTimeoutMs: 300000
# Controls whether Strimzi generates pod disruption budget resources (By default true)
generatePodDisruptionBudget: true
```

## 上書き用 values の例

以下は本番向けの上書き例である。
監視対象 Namespace は事前に作成しておく。
Helm helper は常に `.Release.Namespace`（`strimzi`）も監視対象に加えるため、`kafka` も含める。

```bash
kubectl create namespace kafka
kubectl create namespace kafka-prod
kubectl create namespace kafka-staging
```

期待される出力の例は次のとおりである。

```text
namespace/kafka created
namespace/kafka-prod created
namespace/kafka-staging created
```

```yaml
replicas: 2
watchNamespaces:
  - kafka
  - kafka-prod
  - kafka-staging
watchAnyNamespace: false
logLevel: WARN
fullReconciliationIntervalMs: 120000
resources:
  limits:
    memory: 512Mi
    cpu: 1000m
  requests:
    memory: 512Mi
    cpu: 500m
extraEnvs:
  - name: STRIMZI_LOG_LEVEL
    value: WARN
```

`helm upgrade --install` で適用する（chart バージョンを 1.1.0 に固定する）。
release が無い場合も同じコマンドで初回インストールできる。

```bash
helm upgrade --install strimzi-kafka-operator strimzi/strimzi-kafka-operator \
  --version 1.1.0 -n strimzi -f my-values.yaml
```

期待される出力の例は次のとおりである。

```text
Release "strimzi-kafka-operator" has been upgraded. Happy Helming!
```

## 動作確認

インストール済み chart の values を確認する。

```bash
helm get values strimzi-kafka-operator -n strimzi
```

期待される出力の例は次のとおりである。

```text
USER-SUPPLIED VALUES:
extraEnvs:
- name: STRIMZI_LOG_LEVEL
  value: WARN
fullReconciliationIntervalMs: 120000
logLevel: WARN
replicas: 2
resources:
  limits:
    cpu: 1000m
    memory: 512Mi
  requests:
    cpu: 500m
    memory: 512Mi
watchAnyNamespace: false
watchNamespaces:
- kafka
- kafka-prod
- kafka-staging
```

Helm helper は `.Release.Namespace`（`strimzi`）も監視対象に加える。
上記の例では `STRIMZI_NAMESPACE` は `kafka,kafka-prod,kafka-staging,strimzi` になる。

Deployment の環境変数への反映を確認する。

```bash
kubectl get deployment strimzi-cluster-operator -n strimzi \
  -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}' \
  | grep -E 'STRIMZI_NAMESPACE|STRIMZI_FEATURE_GATES|STRIMZI_FULL'
```

[install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml L41-L88](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml#L41-L88)は次のとおりである。

```yaml
          env:
            - name: STRIMZI_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: STRIMZI_FULL_RECONCILIATION_INTERVAL_MS
              value: "120000"
            - name: STRIMZI_OPERATION_TIMEOUT_MS
              value: "300000"
            - name: STRIMZI_DEFAULT_KAFKA_EXPORTER_IMAGE
              value: quay.io/strimzi/kafka:1.1.0-kafka-4.3.0
            - name: STRIMZI_DEFAULT_CRUISE_CONTROL_IMAGE
              value: quay.io/strimzi/kafka:1.1.0-kafka-4.3.0
            - name: STRIMZI_KAFKA_IMAGES
              value: |
                4.2.0=quay.io/strimzi/kafka:1.1.0-kafka-4.2.0
                4.2.1=quay.io/strimzi/kafka:1.1.0-kafka-4.2.1
                4.3.0=quay.io/strimzi/kafka:1.1.0-kafka-4.3.0
            - name: STRIMZI_KAFKA_CONNECT_IMAGES
              value: |
                4.2.0=quay.io/strimzi/kafka:1.1.0-kafka-4.2.0
                4.2.1=quay.io/strimzi/kafka:1.1.0-kafka-4.2.1
                4.3.0=quay.io/strimzi/kafka:1.1.0-kafka-4.3.0
            - name: STRIMZI_KAFKA_MIRROR_MAKER_2_IMAGES
              value: |
                4.2.0=quay.io/strimzi/kafka:1.1.0-kafka-4.2.0
                4.2.1=quay.io/strimzi/kafka:1.1.0-kafka-4.2.1
                4.3.0=quay.io/strimzi/kafka:1.1.0-kafka-4.3.0
            - name: STRIMZI_DEFAULT_TOPIC_OPERATOR_IMAGE
              value: quay.io/strimzi/operator:1.1.0
            - name: STRIMZI_DEFAULT_USER_OPERATOR_IMAGE
              value: quay.io/strimzi/operator:1.1.0
            - name: STRIMZI_DEFAULT_KAFKA_INIT_IMAGE
              value: quay.io/strimzi/operator:1.1.0
            - name: STRIMZI_DEFAULT_KAFKA_BRIDGE_IMAGE
              value: quay.io/strimzi/kafka-bridge:1.0.0
            - name: STRIMZI_DEFAULT_KANIKO_EXECUTOR_IMAGE
              value: quay.io/strimzi/kaniko-executor:1.1.0
            - name: STRIMZI_DEFAULT_BUILDAH_IMAGE
              value: quay.io/strimzi/buildah:1.1.0
            - name: STRIMZI_DEFAULT_MAVEN_BUILDER
              value: quay.io/strimzi/maven-builder:1.1.0
            - name: STRIMZI_OPERATOR_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: STRIMZI_FEATURE_GATES
              value: ""
```

期待される出力の例（Helm で `watchNamespaces` に3 Namespace を指定した場合）は次のとおりである。

```text
STRIMZI_NAMESPACE=kafka,kafka-prod,kafka-staging,strimzi
STRIMZI_FULL_RECONCILIATION_INTERVAL_MS=120000
STRIMZI_FEATURE_GATES=
```

Helm chart は `STRIMZI_NAMESPACE` に監視対象 Namespace のカンマ区切りリストを設定する。
kubectl apply 方式の Deployment テンプレートでは `valueFrom` で Operator 自身の Namespace のみを参照する。

## まとめ

Helm values で監視範囲、ログ、リコンサイル間隔、機能ゲートを制御する。
オペランドごとのイメージと Operator の `resources` を環境に合わせて上書きする。
`helm get values` と Deployment の env で反映を検証する。

## 関連する章

- [第2章 インストール](../part00-introduction/02-installation.md)
- [第23章 スケーリングとローリング更新](23-scaling-rolling-update.md)
- [第24章 トラブルシューティングと運用](24-troubleshooting.md)
