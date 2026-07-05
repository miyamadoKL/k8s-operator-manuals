# 第20章 Read Replica

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml L820-L859](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L820-L859)
> - [helm/mysql-operator/crds/crd.yaml L307-L323](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L307-L323)

## この章でできるようになること

InnoDBCluster に **Read Replica** を追加し、参照専用の負荷を Group Replication のメンバーから切り離せるようになる。

## 前提

InnoDBCluster が作成済みで ONLINE の状態にあることを前提とする。
インスタンス数の変更手順そのものは第19章で扱う。

## Read Replica とは何か

Read Replica は、Group Replication には参加せず、非同期レプリケーションでプライマリからデータを受け取る読み取り専用のインスタンス群である。
通常のセカンダリと異なり、Read Replica はクラスタの書き込みクォーラムの計算に加わらない。
そのため、参照専用の負荷を大量にさばきたい場合や、地理的に離れた拠点に読み取り専用のレプリカを置きたい場合に、クォーラムの過半数条件を崩さずにインスタンスを増やせる。

InnoDBCluster の `spec.readReplicas` は、この Read Replica の集合を配列として定義する。

[helm/mysql-operator/crds/crd.yaml L820-L859](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L820-L859)

```yaml
readReplicas:
  type: array
  items:
    type: object
    required: ["name", "baseServerId"]
    properties:
      name:
        type: string
      version:
        type: string
        pattern: '^\d+\.\d+\.\d+(-.+)?'
        description: "MySQL Server version"
      baseServerId:
        type: integer
        minimum: 0
        maximum: 4294967195
        default: 0
        description: "Base value for MySQL server_id for instances of the readReplica, if 0 it will be assigned automatically"
      datadirVolumeClaimTemplate:
        type: object
        x-kubernetes-preserve-unknown-fields: true
        description: "Template for a PersistentVolumeClaim, to be used as datadir"
      mycnf:
        type: string
        description: "Custom configuration additions for my.cnf"
      instances:
        type: integer
        minimum: 1
        maximum: 999
        default: 1
        description: "Number of MySQL instances for the set of read replica"
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

`readReplicas` は配列であり、1つの要素が「同じ設定を持つ Read Replica インスタンス群」を表す。
`name` は要素ごとの識別名、`instances` はその要素が持つ Read Replica の台数である。
`baseServerId` を明示しない場合（既定値 `0`）、Operator が自動的に採番する。

## 最小構成の例

以下は、既存の InnoDBCluster に1台の Read Replica を追加する自作例である。

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
  readReplicas:
    - name: reporting
      instances: 1
```

`readReplicas[0].name` の `reporting` は、この要素を指す任意の識別名である。
Operator は要素ごとに専用の StatefulSet（`mycluster-reporting` のような名前）を作成し、そこに属する Pod をプライマリからの非同期レプリケーションで追従させる。

```console
$ kubectl apply -f readreplica-reporting.yaml
innodbcluster.mysql.oracle.com/mycluster configured

$ kubectl get pods -l mysql.oracle.com/cluster=mycluster,mysql.oracle.com/read-replica=reporting
NAME                     READY   STATUS    RESTARTS   AGE
mycluster-reporting-0    2/2     Running   0          2m
```

## Router 経由での参照

Router の `routingOptions.read_only_targets` を設定すると、読み取り専用の接続先に Read Replica を含められる。

[helm/mysql-operator/crds/crd.yaml L307-L323](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L307-L323)

```yaml
routingOptions:
  description: "Set routing options for the cluster"
  type: object
  properties:
    # naming pattern follows Shell's naming documented at
    # https://dev.mysql.com/doc/mysql-shell/8.1/en/innodb-clusterset-router-setroutingoption.html
    # ClusterSet-related options and tags currently not supported
    invalidated_cluster_policy:
      type: string
      enum: ["drop_all", "accept_ro"]
    stats_updates_frequency:
      type: integer
      default: 0
      minimum: 0
    read_only_targets:
      type: string
      enum: ["all", "read_replicas", "secondaries"]
```

`read_only_targets` は3つの値を取り、`all` は Group Replication のセカンダリと Read Replica の両方、`read_replicas` は Read Replica のみ、`secondaries` は Group Replication のセカンダリのみを読み取り専用の接続先とする。
Router の設定そのものの詳細は第11章で扱う。

以下は、読み取り専用の接続を Read Replica に限定する自作例である。

```yaml
spec:
  router:
    routingOptions:
      read_only_targets: "read_replicas"
```

```console
$ kubectl patch innodbcluster mycluster --type=merge -p '{"spec":{"router":{"routingOptions":{"read_only_targets":"read_replicas"}}}}'
innodbcluster.mysql.oracle.com/mycluster patched
```

## 動作確認

Read Replica の Pod が非同期レプリケーションに追従できているかは、対象 Pod 上で `SHOW REPLICA STATUS` を確認する。

```console
$ kubectl exec mycluster-reporting-0 -c mysql -- \
    mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS\G" | grep -E "Replica_IO_Running|Replica_SQL_Running|Seconds_Behind_Source"
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
```

`Replica_IO_Running` と `Replica_SQL_Running` がともに `Yes`、`Seconds_Behind_Source` が0に近ければ、Read Replica はプライマリの更新に追従できている。

## Read Replica を増減する

`readReplicas` 配列の要素内の `instances` を書き換えると、その要素の Read Replica を増減できる。
挙動は第19章で説明した `spec.instances` によるスケーリングと同様で、対応する StatefulSet の `replicas` が更新される。

```console
$ kubectl patch innodbcluster mycluster --type=merge -p '{"spec":{"readReplicas":[{"name":"reporting","instances":2}]}}'
innodbcluster.mysql.oracle.com/mycluster patched
```

## トラブルシューティング

- **Read Replica の Pod が起動しない**：`readReplicas[].datadirVolumeClaimTemplate` を省略した場合、クラスタ本体の `datadirVolumeClaimTemplate` と異なるストレージクラスが既定で使われることがある。PVC の状態は `kubectl get pvc` で確認する。
- **`Seconds_Behind_Source` が増え続ける**：プライマリ側の書き込み量に対して Read Replica の I/O 性能が不足している可能性がある。`readReplicas[].podSpec` でリソース割り当てを見直す。
- **Router 経由の参照が Read Replica に振り分けられない**：`read_only_targets` の設定漏れ、または Router の再起動待ちであることが多い。Router のログ確認は第22章を参照する。

## まとめ

`readReplicas` は Group Replication のクォーラムに参加しない読み取り専用のインスタンス群であり、配列の要素ごとに独立した台数とバージョンを持てる。
Router の `read_only_targets` と組み合わせることで、参照専用の負荷をクラスタ本体から切り離せる。

## 関連する章

- [第19章 スケーリングとアップグレード](19-scaling-upgrade.md)
- [第21章 メトリクスとログ](21-metrics-logs.md)
- [第22章 トラブルシューティングと診断](22-troubleshooting.md)
