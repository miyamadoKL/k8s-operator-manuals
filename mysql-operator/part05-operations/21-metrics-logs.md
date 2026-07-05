# 第21章 メトリクスとログ

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml L346-L398](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L346-L398)
> - [helm/mysql-operator/crds/crd.yaml L634-L819](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L634-L819)

## この章でできるようになること

InnoDBCluster の `metrics` で Prometheus 向けのメトリクス収集を、`logs` で MySQL のログ出力を有効化できるようになる。

## 前提

InnoDBCluster が作成済みであることを前提とする。
Prometheus Operator を併用する場合は、あらかじめクラスタに導入されていることを前提とする。

## メトリクス収集の有効化

InnoDBCluster の `spec.metrics` は、Pod に追加するメトリクス収集用サイドカーの設定を表す。

[helm/mysql-operator/crds/crd.yaml L346-L398](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L346-L398)

```yaml
metrics:
  type: object
  description: "Configuration of a Prometheus-style metrics provider"
  required: ["enable", "image"]
  properties:
    enable:
      type: boolean
      default: false
      description: "Toggle to enable or disable the metrics sidecar"
    image:
      type: string
      description: "Name of an image to be used for the metrics sidecar, if provided metrics will be enabled"
    options:
      type: array
      description: "Options passed to the metrics provider as command line arguments"
      items:
        type: string
    webConfig:
      type: string
      description: "Name of a ConfigMap with a web.config file, if this option is provided a command line option --web.config.file is added"
    tlsSecret:
      type: string
      description: "Name of a Secret with TLS certificate, key and CA, which will be mounted at /tls into the container an can be used from webConfig"
    monitor:
      type: boolean
      description: "Create a ServiceMonitor for Prometheus Operator"
      default: false
    monitorSpec:
      type: object
      x-kubernetes-preserve-unknown-fields: true
      description: "Custom configuration for the ServiceMonitor object"
      default: {}
```

`enable` と `image` が必須であり、`image` には mysqld-exporter 相当のメトリクス収集用イメージを指定する。
`monitor` を `true` にすると、Operator は Prometheus Operator の `ServiceMonitor` を自動生成し、Prometheus 側のスクレイプ対象への登録を省略できる。

以下は、メトリクス収集を有効化し、Prometheus Operator 向けに `ServiceMonitor` も作成する自作例である。

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  router:
    instances: 1
  tlsUseSelfSigned: true
  metrics:
    enable: true
    image: "prom/mysqld-exporter:v0.15.1"
    monitor: true
```

```console
$ kubectl patch innodbcluster mycluster --type=merge -p '{"spec":{"metrics":{"enable":true,"image":"prom/mysqld-exporter:v0.15.1","monitor":true}}}'
innodbcluster.mysql.oracle.com/mycluster patched
```

サイドカーが追加された Pod では、コンテナ数が1つ増える。

```console
$ kubectl get pods mycluster-0 -o jsonpath='{.spec.containers[*].name}'
mysql sidecar metrics

$ kubectl get servicemonitor mycluster -o jsonpath='{.spec.endpoints[0].port}'
metrics
```

`mysql`、`sidecar` に加えて `metrics` コンテナが増えていれば、サイドカーが起動している。
`ServiceMonitor` の存在は `kubectl get servicemonitor` で確認できる。

## ログ収集の有効化

InnoDBCluster の `spec.logs` は、MySQL Server が出力する3種類のログ（エラーログ、一般ログ、スロークエリログ）の有効化と、外部への転送に関する設定を表す。

[helm/mysql-operator/crds/crd.yaml L634-L819](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L634-L819)

```yaml
logs:
  type: object
  properties:
    general:
      type: object
      properties:
        enabled:
          type: boolean
          default: false
          description: "Whether general logging should be enabled"
        collect:
          type: boolean
          default: false
          description: "Whether general logging data should be collected. Implies that the logging should be enabled."
    error:
      type: object
      properties:
        collect:
          type: boolean
          default: false
          description: "Whether error logging data should be collected. Implies that the logging should be enabled. If enabled the error log will be switched to JSON format output"
        verbosity:
          type: integer
          default: 3
          minimum: 1
          maximum: 3
          description: "Log error verbosity. For details, see the MySQL Server --log-error-verbosity documentation."
    slowQuery:
      type: object
      properties:
        enabled:
          type: boolean
          default: false
          description: "Whether slow query logging should be enabled"
        longQueryTime:
          type: number
          minimum: 0
          default: 10
          description: "Long query time threshold"
        collect:
          type: boolean
          default: false
          description: "Whether slow query logging data should be collected. Implies that the logging should be enabled."
    collector:
      type: object
      oneOf:
      - required: ["image", "fluentd"]
      properties:
        image:
          type: string
          description: "Name of an image, including registry and repository, to be used for the log collector sidecar. If provided it needs to be an image for the configured collector type."
```

`error` ログは常に出力されており、`collect` を `true` にすると出力形式が JSON に切り替わる。
`general` ログと `slowQuery` ログは既定で無効であり、`enabled` を `true` にして初めて出力される。
`slowQuery.longQueryTime` は、スロークエリとみなす実行時間のしきい値を秒単位で指定する。

3種類のログのいずれかで `collect` を `true` にすると、Operator は `logs.collector` の設定に基づいてログ収集用のサイドカーを追加する。
`collector` は `image` と `fluentd` の両方を必須とし、本書の執筆時点で選択できる収集方式は fluentd のみである。

以下は、スロークエリログを有効化し、5秒を超えるクエリを対象に、fluentd サイドカーで収集する自作例である。

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
spec:
  secretName: mypwds
  instances: 3
  router:
    instances: 1
  tlsUseSelfSigned: true
  logs:
    slowQuery:
      enabled: true
      longQueryTime: 5
      collect: true
    collector:
      image: "fluent/fluentd:v1.16-1"
      fluentd:
        sinks:
          - name: stdout-sink
            rawConfig: |
              <match **>
                @type stdout
              </match>
```

```console
$ kubectl patch innodbcluster mycluster --type=merge --patch-file=logs-slowquery.yaml
innodbcluster.mysql.oracle.com/mycluster patched
```

## 動作確認

ログ収集サイドカーが追加された Pod は、コンテナ一覧に `logcollector`（`collector.containerName` の既定値）が加わる。

```console
$ kubectl get pods mycluster-0 -o jsonpath='{.spec.containers[*].name}'
mysql sidecar logcollector

$ kubectl logs mycluster-0 -c logcollector --tail=5
```

スロークエリログが実際に記録されているかは、対象 Pod 上でスロークエリログテーブルを確認する。

```console
$ kubectl exec mycluster-0 -c mysql -- \
    mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT start_time, query_time, sql_text FROM mysql.slow_log ORDER BY start_time DESC LIMIT 1\G"
```

## トラブルシューティング

- **`metrics.enable: true` にしてもサイドカーが追加されない**：`image` が未指定だと CRD の `required` 制約に違反し、`kubectl apply` の時点で拒否される。エラーメッセージに `metrics.image` が含まれていないか確認する。
- **`ServiceMonitor` が作成されない**：Prometheus Operator の CRD（`ServiceMonitor`）が未導入のクラスタでは `monitor: true` を指定してもリソースを作成できない。`kubectl get crd servicemonitors.monitoring.coreos.com` で導入状況を確認する。
- **ログ収集サイドカーが `CrashLoopBackOff` になる**：`collector.fluentd.sinks` の `rawConfig` の構文誤りが原因になることが多い。`kubectl logs <Pod> -c logcollector` で fluentd 側のエラーを確認する。

## まとめ

`metrics` は Prometheus 向けのメトリクス収集サイドカーと `ServiceMonitor` の生成を、`logs` はエラーログ、一般ログ、スロークエリログの有効化と fluentd サイドカーによる収集を担う。
いずれも既存の Pod にサイドカーコンテナを追加する形で有効化される。

## 関連する章

- [第20章 Read Replica](20-read-replicas.md)
- [第22章 トラブルシューティングと診断](22-troubleshooting.md)
