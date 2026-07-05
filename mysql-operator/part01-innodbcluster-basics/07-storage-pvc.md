# 第7章 ストレージと PersistentVolumeClaim

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L68-L71](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L68-L71)
> - [samples/sample-cluster-pvc.yaml#L10-L24](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-cluster-pvc.yaml#L10-L24)
> - [mysqloperator/controller/innodbcluster/cluster_objects.py#L511-L552](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L511-L552)

## この章でできるようになること

`datadirVolumeClaimTemplate` を使って、MySQL Server のデータディレクトリに使う PersistentVolumeClaim（PVC）の容量とストレージクラスを指定できるようになる。

## 前提

第6章でサーバー構成（`instances` など）を確認済みであることを前提とする。

## datadirVolumeClaimTemplate フィールド

[helm/mysql-operator/crds/crd.yaml#L68-L71](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L68-L71)

```yaml
                datadirVolumeClaimTemplate:
                  type: object
                  x-kubernetes-preserve-unknown-fields: true
                  description: "Template for a PersistentVolumeClaim, to be used as datadir"
```

`x-kubernetes-preserve-unknown-fields: true` のため、CRD のスキーマとしてはこのフィールドの内部構造を検証しない。
実際には Kubernetes の `PersistentVolumeClaimSpec` と同じ構造（`accessModes`、`resources.requests.storage`、`storageClassName` など）をそのまま書く。

Operator が生成する StatefulSet には、datadir 用の `volumeClaimTemplates` があらかじめ組み込まれている。

[mysqloperator/controller/innodbcluster/cluster_objects.py#L511-L518](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L511-L518)

```yaml
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi
```

`datadirVolumeClaimTemplate` を指定すると、この既定のテンプレートに対してマージパッチが適用される。

[mysqloperator/controller/innodbcluster/cluster_objects.py#L549-L552](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_objects.py#L549-L552)

```python
    if spec.datadirVolumeClaimTemplate:
        print("\t\tAdding datadirVolumeClaimTemplate")
        utils.merge_patch_object(statefulset["spec"]["volumeClaimTemplates"][0]["spec"],
                                 spec.datadirVolumeClaimTemplate, "spec.volumeClaimTemplates[0].spec")
```

つまり `datadirVolumeClaimTemplate` を省略しても `emptyDir` にはならず、既定値の `2Gi`（`ReadWriteOnce`）の PVC がそのまま使われる。
`2Gi` は検証用の小さな値であり、本番運用のデータ容量には通常足りない。
そのため、本番環境では `resources.requests.storage` を実際のデータ量に見合った値へ必ず上書きする。

## マニフェスト例

公式サンプルは、ストレージ容量を指定した InnoDBCluster の例である。

[samples/sample-cluster-pvc.yaml#L10-L24](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-cluster-pvc.yaml#L10-L24)

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: idc-with-custom-config
spec:
  secretName: mypwds
  instances: 3
  router:
    instances: 1
  tlsUseSelfSigned: true
  datadirVolumeClaimTemplate:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 300Gi
```

`storageClassName` は指定されていないため、この例では namespace のデフォルト StorageClass が使われる。
特定のストレージクラスを使いたい場合は、以下のように自分の環境向けに `storageClassName` を追加する。

```yaml
  datadirVolumeClaimTemplate:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 300Gi
```

## 動作確認

InnoDBCluster を適用した後、各インスタンスに紐づく PVC が作成されていることを確認する。

```bash
kubectl get pvc -l "mysql.oracle.com/cluster=idc-with-custom-config"
```

```text
NAME                            STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS
datadir-idc-with-custom-config-0   Bound    pvc-xxxx   300Gi      RWO            standard
datadir-idc-with-custom-config-1   Bound    pvc-yyyy   300Gi      RWO            standard
datadir-idc-with-custom-config-2   Bound    pvc-zzzz   300Gi      RWO            standard
```

`CAPACITY` が `resources.requests.storage` で指定した値と一致していれば、意図通りの PVC が作成されている。

## 容量変更の注意

`datadirVolumeClaimTemplate` は StatefulSet の `volumeClaimTemplates` として使われる。
Kubernetes の StatefulSet は、既存 Pod の `volumeClaimTemplates` を変更しても既存 PVC のサイズを自動追従しない。
そのため、容量を拡張したい場合は、対応する StorageClass が `allowVolumeExpansion: true` であることを確認したうえで、PVC を個別に編集する必要がある。

## まとめ

`datadirVolumeClaimTemplate` は、MySQL Server の datadir に使う PVC のテンプレートである。
CRD としては構造を検証しないため、Kubernetes の `PersistentVolumeClaimSpec` をそのまま記述する。
省略すると既定値の `2Gi` の PVC が使われるため、本番運用では容量を実データ量に見合った値へ必ず上書きする。

## 関連する章

- 前章：[第6章 サーバー構成](06-server-config.md)
- 次章：[第8章 MySQL 設定（mycnf）](08-mycnf.md)
- [第19章 スケーリングとアップグレード](../part05-operations/19-scaling-upgrade.md)
