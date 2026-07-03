# 第10章 監視とメトリクス

> - [api/v1beta2/sparkapplication_types.go L650-L693](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L650-L693)
> - [api/v1beta2/sparkapplication_types.go L284-L310](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L284-L310)
> - [examples/spark-pi-prometheus.yaml L17-L71](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-prometheus.yaml#L17-L71)
> - [examples/spark-pi-prometheus-servlet.yaml L16-L68](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-prometheus-servlet.yaml#L16-L68)

## この章でできるようになること

- **MonitoringSpec** を設定して、Prometheus JMX Exporter で driver・executor のメトリクスを公開できる。
- **Spark UI** の Service・Ingress を設定して、Web UI にアクセスできる。
- Prometheus Servlet を使った代替のメトリクス公開方法を理解できる。

## 前提

- 第4章で SparkApplication の基本構造を理解していること。
- Prometheus の基本概念（メトリクススクレイピング、エンドポイント）を理解していること。

## Prometheus JMX Exporter によるメトリクス公開

**MonitoringSpec** は driver・executor のメトリクスを Prometheus に公開するための設定を提供する。

[api/v1beta2/sparkapplication_types.go の MonitoringSpec](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L650-L693)は次のように定義されている。

```go
// MonitoringSpec defines the monitoring specification.
type MonitoringSpec struct {
	// ExposeDriverMetrics specifies whether to expose metrics on the driver.
	ExposeDriverMetrics bool `json:"exposeDriverMetrics"`
	// ExposeExecutorMetrics specifies whether to expose metrics on the executors.
	ExposeExecutorMetrics bool `json:"exposeExecutorMetrics"`
	// MetricsProperties is the content of a custom metrics.properties for configuring the Spark metric system.
	// ...
	MetricsProperties *string `json:"metricsProperties,omitempty"`
	// MetricsPropertiesFile is the container local path of file metrics.properties for configuring
	// the Spark metric system. If not specified, value /etc/metrics/conf/metrics.properties will be used.
	// ...
	MetricsPropertiesFile *string `json:"metricsPropertiesFile,omitempty"`
	// Prometheus is for configuring the Prometheus JMX exporter.
	// +optional
	Prometheus *PrometheusSpec `json:"prometheus,omitempty"`
}

// PrometheusSpec defines the Prometheus specification when Prometheus is to be used for
// collecting and exposing metrics.
type PrometheusSpec struct {
	// JmxExporterJar is the path to the Prometheus JMX exporter jar in the container.
	JmxExporterJar string `json:"jmxExporterJar"`
	// Port is the port of the HTTP server run by the Prometheus JMX exporter.
	// If not specified, 8090 will be used as the default.
	// ...
	Port *int32 `json:"port,omitempty"`
	// PortName is the port name of prometheus JMX exporter port.
	// ...
	PortName *string `json:"portName,omitempty"`
	// ConfigFile is the path to the custom Prometheus configuration file provided in the Spark image.
	// ConfigFile takes precedence over Configuration, which is shown below.
	// ...
	ConfigFile *string `json:"configFile,omitempty"`
	// Configuration is the content of the Prometheus configuration needed by the Prometheus JMX exporter.
	// ...
	Configuration *string `json:"configuration,omitempty"`
}
```

各フィールドの役割を次の表にまとめる。

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `exposeDriverMetrics` | `bool` | driver のメトリクス公開の可否 |
| `exposeExecutorMetrics` | `bool` | executor のメトリクス公開の可否 |
| `metricsProperties` | `*string` | カスタム `metrics.properties` の内容 |
| `metricsPropertiesFile` | `*string` | `metrics.properties` ファイルのパス（デフォルト `/etc/metrics/conf/metrics.properties`） |
| `prometheus.jmxExporterJar` | `string` | Prometheus JMX Exporter JAR のパス |
| `prometheus.port` | `*int32` | JMX Exporter の HTTP サーバポート（デフォルト `8090`） |
| `prometheus.configFile` | `*string` | Prometheus 設定ファイルのパス |
| `prometheus.configuration` | `*string` | Prometheus 設定の内容 |

### Prometheus JMX Exporter の例

[examples/spark-pi-prometheus.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-prometheus.yaml#L17-L71)は次のとおりである。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: {IMAGE_REGISTRY}/{IMAGE_REPOSITORY}/docker.io/library/spark:4.0.1-gcs-prometheus
  imagePullPolicy: Always
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: local:///opt/spark/examples/jars/spark-examples.jar
  arguments:
  - "100000"
  sparkVersion: 4.0.1
  restartPolicy:
    type: Never
  driver:
    cores: 1
    memory: 512m
    labels:
      version: 4.0.0
    serviceAccount: spark-operator-spark
  executor:
    cores: 1
    instances: 1
    memory: 512m
    labels:
      version: 4.0.0
  monitoring:
    exposeDriverMetrics: true
    exposeExecutorMetrics: true
    prometheus:
      jmxExporterJar: /prometheus/jmx_prometheus_javaagent-0.11.0.jar
      port: 8090
```

`monitoring.exposeDriverMetrics` と `monitoring.exposeExecutorMetrics` を `true` に設定し、`prometheus.jmxExporterJar` に JMX Exporter JAR のパスを指定する。
この例では Prometheus JMX Exporter 同梱のカスタムイメージを使用している。

## Prometheus Servlet による代替方法

Spark 3.0 以降では、JMX Exporter を使わずに Prometheus Servlet でメトリクスを公開する方法がある。
`sparkConf` で Prometheus 用の設定を指定する。

[examples/spark-pi-prometheus-servlet.yaml](https://github.com/kubeflow/spark-operator/blob/v2.5.1/examples/spark-pi-prometheus-servlet.yaml#L16-L68)は次のとおりである。

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-python
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: docker.io/library/spark:4.0.1
  imagePullPolicy: IfNotPresent
  mainApplicationFile: local:///opt/spark/examples/src/main/python/pi.py
  sparkVersion: 4.0.1
  sparkConf:
    # Expose Spark metrics for Prometheus
    "spark.kubernetes.driver.annotation.prometheus.io/scrape": "true"
    "spark.kubernetes.driver.annotation.prometheus.io/path": "/metrics/executors/prometheus/"
    "spark.kubernetes.driver.annotation.prometheus.io/port": "4040"
    "spark.kubernetes.driver.service.annotation.prometheus.io/scrape": "true"
    "spark.kubernetes.driver.service.annotation.prometheus.io/path": "/metrics/driver/prometheus/"
    "spark.kubernetes.driver.service.annotation.prometheus.io/port": "4040"
    "spark.ui.prometheus.enabled": "true"
    "spark.executor.processTreeMetrics.enabled": "true"
    "spark.metrics.conf.*.sink.prometheusServlet.class": "org.apache.spark.metrics.sink.PrometheusServlet"
    "spark.metrics.conf.driver.sink.prometheusServlet.path": "/metrics/driver/prometheus/"
    "spark.metrics.conf.executor.sink.prometheusServlet.path": "/metrics/executors/prometheus/"
  driver:
    cores: 1
    memory: 512m
    serviceAccount: spark-operator-spark
  executor:
    instances: 1
    cores: 1
    memory: 512m
```

`sparkConf` で Prometheus アノテーションと Servlet の設定を指定している。
この方法では JMX Exporter JAR を含むカスタムイメージが不要になる。

## Spark UI

Spark Operator は driver Pod の Spark UI を公開するための Service を自動作成する。
Helm values の `controller.uiService.enable` が `true`（デフォルト）の場合に有効になる。

**SparkUIConfiguration** で Service・Ingress の設定をカスタマイズできる。

[api/v1beta2/sparkapplication_types.go の SparkUIConfiguration](https://github.com/kubeflow/spark-operator/blob/v2.5.1/api/v1beta2/sparkapplication_types.go#L284-L310)は次のように定義されている。

```go
// SparkUIConfiguration is for driver UI specific configuration parameters.
type SparkUIConfiguration struct {
	// ServicePort allows configuring the port at service level that might be different from the targetPort.
	// ...
	ServicePort *int32 `json:"servicePort,omitempty"`
	// ServicePortName allows configuring the name of the service port.
	// ...
	ServicePortName *string `json:"servicePortName,omitempty"`
	// ServiceType allows configuring the type of the service. Defaults to ClusterIP.
	// ...
	ServiceType *corev1.ServiceType `json:"serviceType,omitempty"`
	// ServiceAnnotations is a map of key,value pairs of annotations that might be added to the service object.
	// ...
	ServiceAnnotations map[string]string `json:"serviceAnnotations,omitempty"`
	// ServiceLabels is a map of key,value pairs of labels that might be added to the service object.
	// ...
	ServiceLabels map[string]string `json:"serviceLabels,omitempty"`
	// IngressAnnotations is a map of key,value pairs of annotations that might be added to the ingress object.
	// ...
	IngressAnnotations map[string]string `json:"ingressAnnotations,omitempty"`
	// TlsHosts is useful If we need to declare SSL certificates to the ingress object
	// ...
	IngressTLS []networkingv1.IngressTLS `json:"ingressTLS,omitempty"`
}
```

以下は自作の設定例である。

```yaml
spec:
  sparkUIOptions:
    serviceType: LoadBalancer
    serviceAnnotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
    servicePort: 80
```

この設定では、Spark UI の Service を `LoadBalancer` タイプで作成し、AWS NLB のアノテーションを追加している。

## 動作確認

Prometheus メトリクスが公開されていることを確認する。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/spark-operator/v2.5.1/examples/spark-pi-prometheus-servlet.yaml
```

driver Pod が Running になったら、メトリクスエンドポイントにアクセスする。

```bash
kubectl port-forward spark-pi-python-driver 4040:4040
```

別のターミナルで次のコマンドを実行する。

```bash
curl http://localhost:4040/metrics/driver/prometheus/
```

Prometheus 形式のメトリクスが出力される。

## まとめ

- **MonitoringSpec** で Prometheus JMX Exporter を使ったメトリクス公開を設定できる。
- **Prometheus Servlet** は JMX Exporter JAR を含まない標準イメージでメトリクスを公開する代替方法である。
- **SparkUIConfiguration** で Spark UI の Service タイプ・アノテーション・Ingress をカスタマイズできる。

## 関連する章

- [第4章 SparkApplication の基本構造](../part01-sparkapplication-basics/04-sparkapplication-spec.md)
- [第14章 Helm values リファレンス](../part04-operations/14-helm-values.md)
