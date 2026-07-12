# 第6章 ストレージ設定

> 本章で参照する公式リソース
>
> - [install/cluster-operator/045-Crd-kafkanodepool.yaml L64-L150](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/045-Crd-kafkanodepool.yaml#L64-L150)
> - [examples/kafka/kafka-persistent.yaml L1-L37](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/kafka/kafka-persistent.yaml#L1-L37)
> - [examples/kafka/kafka-ephemeral.yaml L11-L16](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/kafka/kafka-ephemeral.yaml#L11-L16)
> - [examples/kafka/kafka-jbod.yaml L11-L40](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/kafka/kafka-jbod.yaml#L11-L40)

## この章でできるようになること

- `KafkaNodePool` のストレージ type（`ephemeral`、`persistent-claim`、`jbod`）を選定できる。
- JBOD 構成で複数ボリュームを定義できる。
- `kraftMetadata: shared` の意味を説明できる。
- PVC の状態を確認し、ストレージ拡張の制約を理解できる。

## 前提

[第4章 KafkaNodePool とノードロール](04-kafkanodepool.md)でノードプールの基本構造を理解していること。
本章は第3章のオープンクラスタ（`dual-role` 1台の `my-cluster`）を前提とする。
Kubernetes の StorageClass と PersistentVolumeClaim の概念を知っていること。

## ストレージ type の種類

[install/cluster-operator/045-Crd-kafkanodepool.yaml L94-L100](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/045-Crd-kafkanodepool.yaml#L94-L100)によると、`type` には次の 3 値がある。

```yaml
                  type:
                    type: string
                    enum:
                    - ephemeral
                    - persistent-claim
                    - jbod
                    description: "Storage type, must be either 'ephemeral', 'persistent-claim', or 'jbod'."
```

| type | 説明 | 用途 |
|---|---|---|
| `persistent-claim` | 単一 PVC を Pod にマウント | シンプルな本番構成 |
| `ephemeral` | EmptyDir ベースの揮発ストレージ | 検証やテスト |
| `jbod` | 複数ボリュームを JBOD として構成 | 本番の標準的な選択肢 |

CRD の説明では `storage` 全体の type 変更はできない。
一方で `persistent-claim` のサイズ拡張や、JBOD へのボリューム追加と削除はサポートされる。
ディスク構成を根本的に変えるには新しいノードプールの追加など、別途手順が必要になる。

## persistent-claim と jbod

本番では `jbod` と `volumes` 配列を使う例が多い。
[examples/kafka/kafka-persistent.yaml L11-L17](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/kafka/kafka-persistent.yaml#L11-L17)は 1 ボリュームの jbod である。

```yaml
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        kraftMetadata: shared
```

検証環境では揮発ストレージを選べる。
[examples/kafka/kafka-ephemeral.yaml L11-L16](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/kafka/kafka-ephemeral.yaml#L11-L16)は次のとおりである。

```yaml
  storage:
    type: jbod
    volumes:
      - id: 0
        type: ephemeral
        kraftMetadata: shared
```

Pod 削除時にデータは失われる。
本番ワークロードには `persistent-claim` を使う。

## JBOD 複数ディスク

[examples/kafka/kafka-jbod.yaml L11-L40](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/kafka/kafka-jbod.yaml#L11-L40)は、KRaft メタデータ共有ボリュームとパーティションデータ専用ボリュームの 2 ディスク構成である。

```yaml
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        kraftMetadata: shared
---

apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: broker
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        # Indicates that this directory will be used to store Kraft metadata log
        kraftMetadata: shared
      - id: 1
        type: persistent-claim
        size: 100Gi
```

`id` はボリューム識別子であり、jbod では必須である。
`kraftMetadata: shared` はそのボリュームに KRaft メタデータログとパーティションデータを共有して格納することを示す。
制約は個々の storage 定義内であり、1 つの jbod 定義で `kraftMetadata: shared` を持てるボリュームは高々 1 つである。
コントローラープールとブローカープールのそれぞれに設定できる。

その他のボリュームフィールドは次のとおりである。

| フィールド | 説明 |
|---|---|
| `size` | PVC のサイズ（例: `100Gi`） |
| `class` | StorageClass 名 |
| `deleteClaim` | ノード削除時に PVC も削除するか（デフォルト `false`） |
| `selector` | 既存 PV をラベルで選択 |

## ストレージ拡張

`persistent-claim` の `size` を増やすと、Strimzi は PVC の拡張を試みる。
StorageClass が `allowVolumeExpansion: true` である必要がある。
縮小はサポートされない。

`jbod` でボリュームを追加する場合は、新しい `id` のエントリを追加する。
追加したディスクをパーティション配置に使うには、Cruise Control による再割り当てが必要になる。
既存ボリュームの `type` 変更はできない。

## 動作確認

クラスタに紐づく PVC を一覧する。

```bash
kubectl get pvc -l strimzi.io/cluster=my-cluster -n kafka
```

期待される出力の例（第3章の single-node 構成）は次のとおりである。

```text
NAME                              STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-0-my-cluster-dual-role-0     Bound    pvc-aaa    100Gi      RWO            standard       10m
```

分離構成（コントローラー3台とブローカー3台）では PVC が6件になる。

PVC が `Pending` の場合は StorageClass や空き容量を確認する。

```bash
kubectl describe pvc data-0-my-cluster-dual-role-0 -n kafka
```

`Pending` 状態の PVC では `Events` 節に次のようなメッセージが出ることがある。

```text
  Warning  ProvisioningFailed  2m  persistentvolume-controller  storageclass.storage.k8s.io "standard" not found
```

正常に Bound した PVC の `Status` 節は次のとおりである。

```text
Status:        Bound
```

## まとめ

`KafkaNodePool` の `storage` でディスク種別とサイズを宣言する。
本番は `jbod` と `persistent-claim`、検証は `ephemeral` が典型である。
`kraftMetadata: shared` で KRaft メタデータの配置先を指定する。

## 関連する章

- [第4章 KafkaNodePool とノードロール](04-kafkanodepool.md)
