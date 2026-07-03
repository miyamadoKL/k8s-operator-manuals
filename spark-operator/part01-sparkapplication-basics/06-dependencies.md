# 第6章 依存関係とアプリケーションの配布

> - [api/v1beta2/sparkapplication_types.go L72-L88](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L72-L88)
> - [api/v1beta2/sparkapplication_types.go L389-L417](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L389-L417)
> - [examples/spark-pi-configmap.yaml L16-L68](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-configmap.yaml#L16-L68)

## この章でできるようになること

- **Dependencies** 構造体を使って JAR・Python ファイル・Maven パッケージをアプリケーションに追加できる。
- `sparkConf` と `hadoopConf` で Spark・Hadoop の設定プロパティを渡せる。
- ConfigMap を Volume としてマウントして設定ファイルを配布できる。

## 前提

- 第4章で SparkApplication の基本構造を理解していること。
- 第5章で driver・executor の設定を理解していること。

## sparkConf と hadoopConf

**SparkConf** と **HadoopConf** は `spark-submit` の `--conf` オプションに相当する設定プロパティをキー・バリューのマップで渡す。

[api/v1beta2/sparkapplication_types.go の sparkConf・hadoopConf](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L72-L88)の定義は次のとおりである。

```go
	// SparkConf carries user-specified Spark configuration properties as they would use the  "--conf" option in
	// spark-submit.
	// +optional
	SparkConf map[string]string `json:"sparkConf,omitempty"`
	// HadoopConf carries user-specified Hadoop configuration properties as they would use the "--conf" option
	// in spark-submit. The SparkApplication controller automatically adds prefix "spark.hadoop." to Hadoop
	// configuration properties.
	// +optional
	HadoopConf map[string]string `json:"hadoopConf,omitempty"`
```

`sparkConf` に指定したプロパティは `--conf` として `spark-submit` に渡される。
`hadoopConf` に指定したプロパティにはコントローラが自動的に `spark.hadoop.` プレフィックスを付与する。

以下は自作の設定例である。

```yaml
spec:
  sparkConf:
    spark.sql.shuffle.partitions: "10"
    spark.executor.memoryOverhead: "256"
  hadoopConf:
    fs.s3a.endpoint: "https://s3.amazonaws.com"
```

`hadoopConf` の `fs.s3a.endpoint` は `spark.hadoop.fs.s3a.endpoint` として `spark-submit` に渡される。

## Dependencies

**Dependencies** 構造体は Spark アプリケーションが依存する外部リソースを一括で指定する。

[api/v1beta2/sparkapplication_types.go の Dependencies](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L389-L417)は次のように定義されている。

```go
// Dependencies specifies all possible types of dependencies of a Spark application.
type Dependencies struct {
	// Jars is a list of JAR files the Spark application depends on.
	// +optional
	Jars []string `json:"jars,omitempty"`
	// Files is a list of files the Spark application depends on.
	// +optional
	Files []string `json:"files,omitempty"`
	// PyFiles is a list of Python files the Spark application depends on.
	// +optional
	PyFiles []string `json:"pyFiles,omitempty"`
	// Packages is a list of maven coordinates of jars to include on the driver and executor
	// classpaths. This will search the local maven repo, then maven central and any additional
	// remote repositories given by the "repositories" option.
	// Each package should be of the form "groupId:artifactId:version".
	// +optional
	Packages []string `json:"packages,omitempty"`
	// ExcludePackages is a list of "groupId:artifactId", to exclude while resolving the
	// dependencies provided in Packages to avoid dependency conflicts.
	// +optional
	ExcludePackages []string `json:"excludePackages,omitempty"`
	// Repositories is a list of additional remote repositories to search for the maven coordinate
	// given with the "packages" option.
	// +optional
	Repositories []string `json:"repositories,omitempty"`
	// Archives is a list of archives to be extracted into the working directory of each executor.
	// +optional
	Archives []string `json:"archives,omitempty"`
}
```

各フィールドの対応する `spark-submit` オプションと用途を次の表にまとめる。

| フィールド | spark-submit オプション | 用途 |
| --- | --- | --- |
| `jars` | `--jars` | driver・executor のクラスパスに追加する JAR ファイル |
| `files` | `--files` | 各 executor の作業ディレクトリに配置するファイル |
| `pyFiles` | `--py-files` | Python アプリケーションの依存ファイル（`.py`・`.zip`・`.egg`） |
| `packages` | `--packages` | Maven 座標で指定する依存ライブラリ（`groupId:artifactId:version`） |
| `excludePackages` | `--exclude-packages` | 依存解決から除外するパッケージ |
| `repositories` | `--repositories` | Maven の追加リモートリポジトリ |
| `archives` | `--archives` | executor の作業ディレクトリに展開するアーカイブ |

以下は自作のマニフェスト例である。

```yaml
spec:
  deps:
    jars:
    - s3a://my-bucket/libs/my-library.jar
    packages:
    - com.amazonaws:aws-java-sdk-s3:1.12.500
    repositories:
    - https://repo1.maven.org/maven2/
    pyFiles:
    - s3a://my-bucket/libs/udf.py
```

`jars` と `packages` は並行して使用できる。
`packages` は Maven リポジトリから依存ライブラリを自動解決するため、手動で JAR を配置する必要がない。

## ConfigMap による設定ファイルの配布

ConfigMap を Volume としてマウントすると、`log4j.properties` や `core-site.xml` 等の設定ファイルを driver・executor の Pod に配布できる。

[examples/spark-pi-configmap.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-configmap.yaml#L16-L68)はその例である。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-configmap
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
  volumes:
  - name: config-vol
    configMap:
      name: test-configmap
  driver:
    cores: 1
    memory: 512m
    volumeMounts:
    - name: config-vol
      mountPath: /opt/spark/config
    serviceAccount: spark-operator-spark
  executor:
    instances: 1
    cores: 1
    memory: 512m
    volumeMounts:
    - name: config-vol
      mountPath: /opt/spark/config
```

`spec.volumes` で ConfigMap を Volume として定義し、`driver.volumeMounts` と `executor.volumeMounts` で同じ Volume をマウントする。
driver と executor の両方に同じ設定ファイルを配布できる。

事前に ConfigMap を作成しておく必要がある。

```bash
kubectl create configmap test-configmap --from-file=spark-defaults.conf=/dev/null
```

```text
configmap/test-configmap created
```

その後、SparkApplication を適用する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi-configmap.yaml
```

```text
sparkapplication.sparkoperator.k8s.io/spark-pi-configmap created
```

Pod 内のマウント先を確認する。

```bash
kubectl exec spark-pi-configmap-driver -- ls /opt/spark/config
```

ConfigMap に含まれるファイルが `/opt/spark/config` 以下に配置されていることを確認できる。

## まとめ

- `sparkConf` と `hadoopConf` で Spark・Hadoop の設定プロパティを渡せる。`hadoopConf` には自動的に `spark.hadoop.` プレフィックスが付与される。
- `deps` フィールドで JAR・Python ファイル・Maven パッケージ等の依存リソースを一括指定できる。
- ConfigMap を Volume としてマウントすると、設定ファイルを driver・executor の Pod に配布できる。

## 関連する章

- [第4章 SparkApplication の基本構造](04-sparkapplication-spec.md)
- [第5章 driver と executor の設定](05-driver-executor.md)
- [第8章 Volume と Pod テンプレート](../part02-sparkapplication-advanced/08-volumes-podtemplate.md)
