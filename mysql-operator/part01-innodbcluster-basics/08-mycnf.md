# 第8章 MySQL 設定（mycnf）

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L72-L74](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L72-L74)
> - [samples/sample-cluster-mycnf.yaml#L11-L26](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-cluster-mycnf.yaml#L11-L26)
> - [mysqloperator/controller/innodbcluster/cluster_api.py#L1478-L1482](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_api.py#L1478-L1482)

## この章でできるようになること

`mycnf` フィールドで MySQL Server に独自の設定変数を追加できるようになる。
あわせて、Operator がこのフィールドの内容をどこまで検証しているかを理解し、記述ミスに気づけるようになる。

## 前提

第7章でストレージ設定を確認済みであることを前提とする。

## mycnf フィールド

[helm/mysql-operator/crds/crd.yaml#L72-L74](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L72-L74)

```yaml
                mycnf:
                  type: string
                  description: "Custom configuration additions for my.cnf"
```

`mycnf` は単なる文字列フィールドである。
ここに書いた内容は、MySQL Server が読み込む `my.cnf` に追加設定として組み込まれる。

## マニフェスト例

公式サンプルは、`innodb_buffer_pool_size` と `innodb_log_file_size` を追加する例である。

[samples/sample-cluster-mycnf.yaml#L11-L26](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/samples/sample-cluster-mycnf.yaml#L11-L26)

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

  mycnf: |
    [mysqld]
    innodb_buffer_pool_size=200M
    innodb_log_file_size=2G
```

YAML のブロックスカラー（`|`）を使い、複数行の設定をそのまま `mycnf` に渡す。
`[mysqld]` セクション見出しを含めて記述する必要がある。

## Operator は内容を検証しない

`mycnf` の中身は、YAML としての構文チェックはあっても、MySQL の設定値として正しいかどうかは検証されない。
唯一の例外は `[mysqld]` セクション見出しの有無であり、これが無い場合は警告が出るだけで、クラスタの作成は止まらない。

[mysqloperator/controller/innodbcluster/cluster_api.py#L1478-L1482](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/innodbcluster/cluster_api.py#L1478-L1482)

```python
        if self.mycnf:
            if "[mysqld]" not in self.mycnf:
                logger.warning(
                    "spec.mycnf data does not contain a [mysqld] line")
```

存在しない変数名やタイプミスをそのまま `mycnf` に書いても、Operator はエラーにしない。
その場合、MySQL Server プロセス自身が起動時にエラーを出し、Pod が `CrashLoopBackOff` になって初めて問題に気づくことになる。

## 動作確認

`mycnf` を反映したクラスタで、設定値が実際に読み込まれているかを確認する。

```bash
kubectl exec idc-with-custom-config-0 -c mysql -- mysql -uroot -p"$(kubectl get secret mypwds -o jsonpath='{.data.rootPassword}' | base64 -d)" -N -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size'"
```

```text
innodb_buffer_pool_size	209715200
```

`209715200` バイトは `200M` に相当する。
値が一致していれば `mycnf` の設定が反映されている。

## トラブルシューティング

`mycnf` の変更を適用しても値が反映されない場合、まず対象 Pod が再起動されたかを確認する。
`mycnf` は起動時に読み込まれる設定であるため、`innodb_buffer_pool_size` のような静的変数は Pod の再起動を経て初めて反映される。

```bash
kubectl get pods -l "mysql.oracle.com/cluster=idc-with-custom-config"
```

`RESTARTS` 列が増えていない場合は、StatefulSet のローリング更新が完了していない可能性がある。

## まとめ

`mycnf` は MySQL Server への追加設定を渡すための自由記述フィールドである。
`[mysqld]` セクション見出しの有無だけは警告対象になるが、設定値そのものの正しさは Operator が検証しない。
記述ミスは MySQL Server の起動失敗として現れるため、変更後は必ず Pod の状態を確認する。

## 関連する章

- 前章：[第7章 ストレージと PersistentVolumeClaim](07-storage-pvc.md)
- 次章：[第9章 Pod 仕様のカスタマイズ（podSpec）](09-podspec.md)
