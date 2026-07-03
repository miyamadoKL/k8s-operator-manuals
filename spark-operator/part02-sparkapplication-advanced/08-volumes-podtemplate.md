# 第8章 Volume と Pod テンプレート

> - [api/v1beta2/sparkapplication_types.go L89-L91](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L89-L91)
> - [api/v1beta2/sparkapplication_types.go L421-L430](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L421-L430)
> - [examples/spark-pi-emptydir.yaml L16-L73](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-emptydir.yaml#L16-L73)
> - [examples/spark-pi-pod-template.yaml L52-L217](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-pod-template.yaml#L52-L217)

## この章でできるようになること

- `spec.volumes` と `volumeMounts` で driver・executor Pod に Volume をマウントできる。
- `template` フィールドで Pod テンプレートを指定し、Spark の標準設定では表現できない Pod 設定を行える。

## 前提

- 第5章で driver・executor の基本設定（`cores`・`memory`・`labels` 等）を理解していること。
- 第6章で ConfigMap を Volume としてマウントする方法を経験していること。

## Volume のマウント

**SparkApplicationSpec** の `volumes` フィールドで Pod レベルの Volume を定義し、`driver.volumeMounts`・`executor.volumeMounts` で各コンテナにマウントする。

[api/v1beta2/sparkapplication_types.go の volumes](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L89-L91)の定義は次のとおりである。

```go
	// Volumes is the list of Kubernetes volumes that can be mounted by the driver and/or executors.
	// +optional
	Volumes []corev1.Volume `json:"volumes,omitempty"`
```

`corev1.Volume` 型であるため、`emptyDir`・`configMap`・`secret`・`persistentVolumeClaim` 等の任意の Volume 種別を使用できる。

### emptyDir の例

[examples/spark-pi-emptydir.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-emptydir.yaml#L16-L73)は `emptyDir` Volume を使った例である。

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
  volumes:
    - name: spark-local-dir-1
      emptyDir:
        medium: Memory
        sizeLimit: 1Gi
  driver:
    cores: 1
    memory: 512m
    serviceAccount: spark-operator-spark
    volumeMounts:
      - mountPath: /datadir
        name: spark-local-dir-1
  executor:
    instances: 1
    cores: 1
    memory: 512m
    volumeMounts:
      - mountPath: /datadir
        name: spark-local-dir-1
```

`medium: Memory` を指定すると、tmpfs としてメモリ上に作成される。
`sizeLimit: 1Gi` で上限を設定できる。
この Volume は Spark の shuffle ディレクトリや一時ファイルの保存先に使用できる。

## Pod テンプレート

**SparkPodSpec** の `template` フィールドで Pod テンプレートを指定すると、Spark の標準設定では表現できない Pod レベルの設定を行える。

[api/v1beta2/sparkapplication_types.go の template](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L421-L430)の定義は次のとおりである。

```go
	// Template is a pod template that can be used to define the driver or executor pod configurations that Spark configurations do not support.
	// Spark version >= 3.0.0 is required.
	// Ref: https://spark.apache.org/docs/latest/running-on-kubernetes.html#pod-template.
	// +optional
	// +kubebuilder:validation:Schemaless
	// +kubebuilder:validation:Type:=object
	// +kubebuilder:pruning:PreserveUnknownFields
	Template *corev1.PodTemplateSpec `json:"template,omitempty"`
```

Pod テンプレートで指定できる設定の例を次の表にまとめる。

| 設定項目 | 説明 |
| --- | --- |
| `containers[].env` | 環境変数（ConfigMap・Secret からの参照を含む） |
| `containers[].envFrom` | ConfigMap・Secret からの一括環境変数注入 |
| `containers[].ports` | 追加のポート定義 |
| `containers[].volumeMounts` | Volume のマウント |
| `nodeSelector` | Node Selector |
| `affinity` | Affinity・AntiAffinity |
| `tolerations` | Taint の Toleration |
| `serviceAccountName` | ServiceAccount 名 |
| `securityContext` | Pod レベルのセキュリティコンテキスト |

### Pod テンプレートの例

[examples/spark-pi-pod-template.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-pod-template.yaml#L52-L217)は Pod テンプレートを使った例である。SparkApplication の部分だけを抽出して示す。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-pod-template
  namespace: default
spec:
  type: Scala
  mode: cluster
  sparkVersion: 4.0.1
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  mainClass: org.apache.spark.examples.SparkPi
  arguments:
  - "10000"
  driver:
    template:
      metadata:
        labels:
          spark.apache.org/version: 4.0.0
        annotations:
          spark.apache.org/version: 4.0.0
      spec:
        containers:
        - name: spark-kubernetes-driver
          env:
          - name: KEY0
            value: VALUE0
          - name: KEY1
            valueFrom:
              configMapKeyRef:
                name: test-configmap
                key: KEY1
          - name: KEY2
            valueFrom:
              secretKeyRef:
                name: test-secret
                key: KEY2
          envFrom:
          - configMapRef:
              name: test-configmap-2
          - secretRef:
              name: test-secret-2
          ports:
          - name: custom-port
            containerPort: 12345
            protocol: TCP
        nodeSelector:
          kubernetes.io/os: linux
        affinity:
          podAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    spark-app-name: spark-pi-pod-template
                topologyKey: kubernetes.io/hostname
        tolerations:
        - operator: Exists
          effect: NoSchedule
        serviceAccountName: spark-operator-spark
    cores: 1
    coreLimit: "1"
    memory: 512m
    memoryOverhead: 512m
  executor:
    instances: 1
    template:
      metadata:
        labels:
          spark.apache.org/version: 4.0.0
      spec:
        containers:
        - name: spark-kubernetes-executor
          env:
          - name: KEY0
            value: VALUE0
        nodeSelector:
          kubernetes.io/os: linux
        tolerations:
        - operator: Exists
          effect: NoSchedule
    cores: 1
    coreLimit: 1500m
    memory: 1g
    memoryOverhead: 512m
```

driver の Pod テンプレートでは、ConfigMap・Secret からの環境変数注入、カスタムポートの定義、Node Selector、Pod Affinity、Toleration を指定している。

## Pod テンプレートでの CPU・メモリ指定の注意点

Pod テンプレートの `containers[].resources` で CPU・メモリの要求量や上限を指定しても、Spark 側で上書きされるため期待通りに動作しない。
CPU・メモリは必ず `spec.driver.cores`・`spec.driver.memory`・`spec.executor.cores`・`spec.executor.memory` 等の SparkApplication レベルのフィールドで指定する。

## 動作確認

emptyDir Volume を使ったマニフェストを適用する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi-emptydir.yaml
```

```text
sparkapplication.sparkoperator.k8s.io/spark-pi created
```

driver Pod に Volume がマウントされていることを確認する。

```bash
kubectl exec spark-pi-driver -- df -h /datadir
```

```text
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           1.0G     0  1.0G   0% /datadir
```

`medium: Memory` により tmpfs としてマウントされていることを確認できる。

## まとめ

- `spec.volumes` で Volume を定義し、`driver.volumeMounts`・`executor.volumeMounts` でマウントする。
- `template` フィールドで Pod テンプレートを指定すると、環境変数・Node Selector・Affinity・Toleration 等の Pod レベルの設定を行える。
- Pod テンプレートで CPU・メモリを指定しても Spark 側で上書きされる。リソース量は `cores`・`memory` 等の SparkApplication レベルのフィールドで指定する。

## 関連する章

- [第5章 driver と executor の設定](../part01-sparkapplication-basics/05-driver-executor.md)
- [第6章 依存関係とアプリケーションの配布](../part01-sparkapplication-basics/06-dependencies.md)
