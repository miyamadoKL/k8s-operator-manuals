# 第16章 オンデマンドバックアップ（MySQLBackup）

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L921-L1006](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L921-L1006)（`MySQLBackup` の `spec`）
> - [helm/mysql-operator/crds/crd.yaml#L1007-L1053](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L1007-L1053)（`MySQLBackup` の `status` と表示列）
> - [mysqloperator/controller/backup/operator_backup.py#L17-L36](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/operator_backup.py#L17-L36)（作成時に Job を生成する処理）
> - [mysqloperator/backup_main.py#L119-L131](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/backup_main.py#L119-L131)（バックアップ完了時に書き込まれる `status` の内容、PVC 保存の場合）
> - [mysqloperator/controller/backup/backup_api.py#L320-L363](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/backup_api.py#L320-L363)（`status.status` に `Running` / `Completed` / `Error` を設定する処理）
> - [mysqloperator/controller/backup/backup_api.py#L191](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/backup_api.py#L191)（`deleteBackupData` が未使用であることを示す実装）
> - [mysqloperator/controller/backup/operator_backup.py#L40](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/operator_backup.py#L40)（`MySQLBackup` 削除時のデータ削除処理が未実装であることを示す TODO）

## この章でできるようになること

`MySQLBackup` オブジェクトを作成して手動でバックアップを1回実行し、その進行状況と結果を `status` から確認できるようになる。

## 前提

第15章「[バックアップの概念とプロファイル](15-backup-concepts.md)」の内容を理解していること。
対象の InnoDBCluster に `backupProfiles` を1件以上登録済みであるか、実行時にインラインでプロファイルを指定できる状態であること。

### MySQLBackup の主要フィールド

[helm/mysql-operator/crds/crd.yaml#L921-L1006](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L921-L1006)は `MySQLBackup` の `spec` 定義である。

```yaml
            spec:
              type: object
              required: ["clusterName"]
              properties:
                clusterName:
                  type: string
                backupProfileName:
                  type: string
                backupProfile:
                  type: object
                  description: "backupProfile specification if backupProfileName is not specified"
                  x-kubernetes-preserve-unknown-fields: true
                  properties:
                    podAnnotations:
                      type: object
                      x-kubernetes-preserve-unknown-fields: true
                    podLabels:
                      type: object
                      x-kubernetes-preserve-unknown-fields: true
                    dumpInstance:
                      type: object
                      properties:
                        dumpOptions:
                          type: object
                          description: "A dictionary of key-value pairs passed directly to MySQL Shell's DumpInstance()"
                          x-kubernetes-preserve-unknown-fields: true
                        storage:
                          type: object
                          properties:
                            ociObjectStorage:
                              type: object
                              required: ["bucketName", "credentials"]
                              properties:
                                bucketName:
                                  type: string
                                  description: "Name of the OCI bucket where backup is stored"
                                prefix:
                                  type: string
                                  description: "Path in bucket where backup is stored"
                                credentials:
                                  type: string
                                  description: "Name of a Secret with data for accessing the bucket"
                            s3:
                              type: object
                              required: ["bucketName", "config"]
                              properties:
                                bucketName:
                                  type: string
                                  description: "Name of the S3 bucket where the dump is stored"
                                prefix:
                                  type: string
                                  description: "Path in the bucket where the dump files are stored"
                                config:
                                  type: string
                                  description: "Name of a Secret with S3 configuration and credentials"
                                profile:
                                  type: string
                                  default: ""
                                  description: "Profile being used in configuration files"
                                endpoint:
                                  type: string
                                  description: "Override endpoint URL"
                            azure:
                              type: object
                              required: ["containerName", "config"]
                              properties:
                                containerName:
                                  type: string
                                  description: "Name of the Azure  BLOB Storage container where the dump is stored"
                                prefix:
                                  type: string
                                  description: "Path in the container where the dump files are stored"
                                config:
                                  type: string
                                  description: "Name of a Secret with Azure BLOB Storage configuration and credentials"
                            persistentVolumeClaim:
                              type: object
                              description : "Specification of the PVC to be used. Used 'as is' in pod executing the backup."
                              x-kubernetes-preserve-unknown-fields: true
                          x-kubernetes-preserve-unknown-fields: true
                addTimestampToBackupDirectory:
                  type: boolean
                  default: true
                deleteBackupData:
                  type: boolean
                  default: false
```

主要フィールドをまとめると次のとおりである。

| フィールド | 必須 | 説明 |
|---|---|---|
| `clusterName` | 必須 | バックアップ対象の InnoDBCluster 名 |
| `backupProfileName` | 任意 | 対象クラスタに登録済みの `backupProfiles` から名前で参照する |
| `backupProfile` | 任意 | `backupProfileName` を指定しない場合に、プロファイル定義をインラインで書く |
| `addTimestampToBackupDirectory` | 任意（既定値 `true`） | バックアップ出力先のディレクトリ名にタイムスタンプを付加するか |
| `deleteBackupData` | 任意（既定値 `false`） | フィールドは定義されているが、8.4.9-2.1.11 の実装では未使用である（後述） |

`backupProfileName` と `backupProfile` はどちらか一方を指定する。
`clusterName` だけは必須であり、これによって operator がどの InnoDBCluster のバックアップ用 Secret とイメージを使うかを決める。

`deleteBackupData` は、名前だけを見ると `MySQLBackup` オブジェクトを削除したときに保存先のバックアップデータ本体も削除する設定に見える。
しかし 8.4.9-2.1.11 の実装では、このフィールドはどこからも参照されない。
[mysqloperator/controller/backup/backup_api.py#L191](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/backup_api.py#L191)は、`MySQLBackupSpec` がこのフィールドを `unused` と明記していることを示す。

```python
        self.deleteBackupData: bool = False # unused
```

`MySQLBackup` オブジェクトが削除されたときの処理を見ても、削除に対応するコードは実装されていない。
[mysqloperator/controller/backup/operator_backup.py#L40](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/operator_backup.py#L40)には、削除時にデータ削除用の Job を作る処理が TODO として残っている。

```python
# TODO create a job to delete the data when the job is deleted
```

したがって `deleteBackupData` を `true` にしても、保存先（PVC や OCI/S3/Azure のバケット）に置かれたバックアップデータは自動的には削除されない。
保存先のデータを削除したい場合は、PVC の削除や OCI/S3 バケット側のライフサイクルポリシーなど、保存先そのものの仕組みで運用者が管理する必要がある。

### オンデマンドバックアップを実行する

以下は、第15章で登録したプロファイル `daily-dump-to-pvc` を参照して手動バックアップを実行する例である。

```yaml
apiVersion: mysql.oracle.com/v2
kind: MySQLBackup
metadata:
  name: mycluster-manual-backup
spec:
  clusterName: mycluster
  backupProfileName: daily-dump-to-pvc
```

このマニフェストを適用すると、operator が `on_mysqlbackup_create` ハンドラで反応し、対象クラスタの Secondary（存在しなければ Primary）インスタンスに対してダンプを実行する Job を1つ生成する。

```bash
kubectl apply -f mysqlbackup-manual.yaml
```

```text
mysqlbackup.mysql.oracle.com/mycluster-manual-backup created
```

Job が生成されたことと、その実行状況は次のコマンドで確認する。

```bash
kubectl get jobs -l mysql.oracle.com/cluster=mycluster
```

```text
NAME                                        COMPLETIONS   DURATION   AGE
mycluster-manual-backup-241203120000        1/1           38s        1m
```

`addTimestampToBackupDirectory` の既定値は `true` であるため、Job 名と出力ディレクトリ名にはタイムスタンプが付く。
毎回同じディレクトリに上書きしたい場合は、この値を `false` にする。

### status からバックアップ結果を確認する

バックアップが完了すると、Job 内の `backup_main.py` が `MySQLBackup` オブジェクトの `status` を更新する。
[helm/mysql-operator/crds/crd.yaml#L1007-L1035](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L1007-L1035)は `status` の定義である。

```yaml
            status:
              type: object
              properties:
                status:
                  type: string
                startTime:
                  type: string
                completionTime:
                  type: string
                elapsedTime:
                  type: string
                output:
                  type: string
                method:
                  type: string
                source:
                  type: string
                bucket:
                  type: string
                ociTenancy:
                  type: string
                container:
                  type: string
                spaceAvailable:
                  type: string
                size:
                  type: string
                message:
                  type: string
```

主なフィールドは次のとおりである。

| フィールド | 説明 |
|---|---|
| `status` | 実行結果。`Running` / `Completed` / `Error` のいずれかを取る |
| `method` | 実際に使われた方式（例：`dump-instance/volume`） |
| `source` | ダンプ元インスタンスの接続情報 |
| `startTime` / `completionTime` / `elapsedTime` | 実行時刻と所要時間 |
| `output` | 出力されたディレクトリまたはファイルの名前 |
| `size` | 出力データのサイズ |

`status.status` は、Job が開始した時点で `Running` になり、ダンプが成功すると `Completed`、失敗すると `Error` に遷移する。
[mysqloperator/controller/backup/backup_api.py#L320-L363](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/backup_api.py#L320-L363)は、`MySQLBackup` の `set_started` / `set_succeeded` / `set_failed` がそれぞれの値を書き込む処理であり、ダンプ成功時に呼ばれる `set_succeeded` は次のように `status.status` を `Completed` にする。

```python
        patch = {"status": {
            "status": "Completed",
            "startTime": start_time,
            "completionTime": end_time,
            "elapsedTime": f"{int(hours):02}:{int(minutes):02}:{int(seconds):02}",
            "output": backup_name
        }}
```

```bash
kubectl get mysqlbackup mycluster-manual-backup -o jsonpath='{.status.status}{"\n"}{.status.output}{"\n"}'
```

```text
Completed
mycluster-manual-backup-241203120000
```

`kubectl get mysqlbackup` を列表示で実行すると、`Cluster` / `Status` / `Output` / `Age` の列がそのまま確認できる。
これは CRD の `additionalPrinterColumns` として定義されている表示列である。

```bash
kubectl get mysqlbackup mycluster-manual-backup
```

```text
NAME                       CLUSTER     STATUS      OUTPUT                                    AGE
mycluster-manual-backup    mycluster   Completed   mycluster-manual-backup-241203120000       2m
```

## トラブルシューティング

`status.status` が `Completed` にならず Job が失敗する場合は、まず Job の Pod ログを確認する。

```bash
kubectl logs job/mycluster-manual-backup-241203120000
```

保存先が OCI、S3、Azure の場合、失敗の多くは `credentials` / `config` で指定した Secret の内容誤りに起因する。
保存先が PVC の場合は、PVC が対象クラスタと同じ Namespace に存在し、書き込み可能であることを確認する。

## まとめ

- `MySQLBackup` は `clusterName` と `backupProfileName`（または `backupProfile`）を指定するだけで、既存のプロファイルを使った手動バックアップを1回実行できる。
- 実行結果は `status` の `status`（`Running` / `Completed` / `Error`）/ `output` / `size` などから確認できる。
- `addTimestampToBackupDirectory` を `false` にすると、毎回同じ出力先を上書きする運用にできる。
- `deleteBackupData` は 8.4.9-2.1.11 では未実装であり、保存先のバックアップデータ本体は自動削除されない。削除は保存先側の仕組みで運用者が行う。

## 関連する章

- [第15章 バックアップの概念とプロファイル](15-backup-concepts.md)
- [第17章 スケジュールバックアップ](17-scheduled-backup.md)
- [第18章 リストア](18-restore.md)
