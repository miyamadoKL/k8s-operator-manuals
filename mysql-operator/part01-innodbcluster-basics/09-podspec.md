# 第9章 Pod 仕様のカスタマイズ（podSpec）

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L81-L89](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L81-L89)
> - [mysqloperator/controller/innodbcluster/cluster_objects.py#L523-L547](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L523-L547)
> - [samples/sample-cluster-podspec-resources.yaml#L12-L33](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-cluster-podspec-resources.yaml#L12-L33)
> - [samples/sample-cluster-podspec-node-selector.yaml#L12-L26](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-cluster-podspec-node-selector.yaml#L12-L26)

## この章でできるようになること

`podSpec`、`podAnnotations`、`podLabels` を使って、Operator が生成する MySQL Server Pod に対して、リソース制限やスケジューリング制約などの Kubernetes 標準の設定を追加できるようになる。

## 前提

第8章で `mycnf` によるサーバー設定を確認済みであることを前提とする。

## podSpec / podAnnotations / podLabels フィールド

[helm/mysql-operator/crds/crd.yaml#L81-L89](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L81-L89)

```yaml
                podSpec:
                  type: object
                  x-kubernetes-preserve-unknown-fields: true
                podAnnotations:
                  type: object
                  x-kubernetes-preserve-unknown-fields: true
                podLabels:
                  type: object
                  x-kubernetes-preserve-unknown-fields: true
```

3フィールドとも `x-kubernetes-preserve-unknown-fields: true` であり、CRD としては内部構造を検証しない。
`podSpec` には Kubernetes の Pod の `spec`（`PodSpec`）と同じ構造を、`podAnnotations` と `podLabels` にはそれぞれ annotation と label のキーバリューを書く。

## マージの挙動

Operator は、まず自身が組み立てた StatefulSet の Pod テンプレートを用意し、そこに利用者が指定した `podSpec` をマージパッチとして適用する。

[mysqloperator/controller/innodbcluster/cluster_objects.py#L531-L532](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L531-L532)

```python
    if len(metadata):
        utils.merge_patch_object(statefulset["spec"]["template"], {"metadata" : metadata })
```

[mysqloperator/controller/innodbcluster/cluster_objects.py#L544-L547](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L544-L547)

```python
    if spec.podSpec:
        print("\t\tAdding podSpec")
        utils.merge_patch_object(statefulset["spec"]["template"]["spec"],
                                 spec.podSpec, "spec.podSpec")
```

マージパッチであるため、`podSpec.containers` に `name: mysql` のコンテナを指定すると、Operator が組み立てた `mysql` コンテナの定義に対して差分だけが上書きされる。
コンテナ全体を置き換えるのではなく、指定したキー（`resources` など）だけが追加または上書きされる点に注意する。

## マニフェスト例：リソース制限

公式サンプルは、`mysql` コンテナに CPU とメモリの requests / limits を設定する例である。

[samples/sample-cluster-podspec-resources.yaml#L12-L33](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-cluster-podspec-resources.yaml#L12-L33)

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: idc-with-resources
spec:
  secretName: mypwds
  instances: 3
  router:
    instances: 1
  tlsUseSelfSigned: true

  podSpec:
    containers:
    - name: mysql
      resources:
        requests:
          memory: "2048Mi"
          cpu: "1800m"
        limits:
          memory: "8192Mi"
          cpu: "3600m"
```

動作確認として、Pod に設定されたリソース制限を確認する。

```bash
kubectl get pod idc-with-resources-0 -o jsonpath='{.spec.containers[?(@.name=="mysql")].resources}'
```

```text
{"limits":{"cpu":"3600m","memory":"8192Mi"},"requests":{"cpu":"1800m","memory":"2048Mi"}}
```

指定した値がそのまま反映されていれば成功である。

## マニフェスト例：nodeSelector

`podSpec` は `containers` 以外の `PodSpec` フィールドにも使える。
公式サンプルは、`nodeSelector` で特定ラベルを持つノードにのみスケジュールする例である。

[samples/sample-cluster-podspec-node-selector.yaml#L12-L26](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-cluster-podspec-node-selector.yaml#L12-L26)

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: idc-with-selector
spec:
  secretName: mypwds
  instances: 3
  router:
    instances: 1
  tlsUseSelfSigned: true

  podSpec:
    nodeSelector:
      failure-domain.beta.kubernetes.io/zone: US-ASHBURN-AD-1
```

動作確認として、Pod がスケジュールされたノードを確認する。

```bash
kubectl get pod idc-with-selector-0 -o jsonpath='{.spec.nodeSelector}'
```

```text
{"failure-domain.beta.kubernetes.io/zone":"US-ASHBURN-AD-1"}
```

## podAnnotations と podLabels

`podAnnotations` と `podLabels` は、`podSpec` とは別に、Pod の `metadata.annotations` と `metadata.labels` にキーを追加する。
Prometheus のスクレイプ設定など、Pod の annotation を参照する外部ツールと連携するときに使う。

```yaml
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9104"
```

これは自作の例であり、監視ツール側の設定と合わせて値を決める必要がある。

## まとめ

`podSpec` は Operator が組み立てる Pod テンプレートへのマージパッチとして働き、`resources`、`nodeSelector` など Kubernetes 標準の `PodSpec` フィールドをそのまま指定できる。
`podAnnotations` と `podLabels` は、Pod の annotation と label だけを追加したい場合に使う。
いずれも CRD としては内部構造を検証しないため、記述ミスは Kubernetes API 側のバリデーションエラーとして現れる。

## 関連する章

- 前章：[第8章 MySQL 設定（mycnf）](08-mycnf.md)
- 次章：[第10章 Service と接続エンドポイント](../part02-networking/10-service-connection.md)
- [第21章 メトリクスとログ](../part05-operations/21-metrics-logs.md)
