# 第17章 スケジュールバックアップ

> 本章で参照する公式リソース
>
> - [helm/mysql-operator/crds/crd.yaml#L534-L633](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L534-L633)（InnoDBCluster の `backupSchedules`）
> - [mysqloperator/controller/backup/backup_objects.py#L193-L222](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/backup_objects.py#L193-L222)（スケジュールごとの CronJob 名生成とテンプレート適用）
> - [mysqloperator/controller/backup/backup_objects.py#L225-L284](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/backup_objects.py#L225-L284)（CronJob テンプレート本体）
> - [mysqloperator/controller/backup/backup_api.py#L124](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/backup_api.py#L124)（`backupSchedules` の `deleteBackupData` が未使用であることを示す実装）
> - [mysqloperator/controller/backup/backup_api.py#L337](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/mysqloperator/controller/backup/backup_api.py#L337)（バックアップ成功時に `status.status` が `Completed` になる処理）

## この章でできるようになること

InnoDBCluster の `spec.backupSchedules` で定期実行のバックアップを設定し、生成される CronJob と、実行ごとに作られる `MySQLBackup` を確認できるようになる。

## 前提

第16章「[オンデマンドバックアップ（MySQLBackup）](16-ondemand-backup.md)」で、バックアッププロファイルを使った手動バックアップの流れを理解していること。

### backupSchedules の主要フィールド

InnoDBCluster の `spec.backupSchedules` は配列であり、各エントリが1つの定期バックアップを表す。
[helm/mysql-operator/crds/crd.yaml#L542-L633](https://github.com/mysql/mysql-operator/blob/8.4.9-2.1.11/helm/mysql-operator/crds/crd.yaml#L542-L633)は、`backupProfile` インラインフィールドを除いた各エントリの定義である。

```yaml
                      name:
                        type: string
                        description: "Name of the backup schedule"
                      schedule:
                        type: string
                        description: "The schedule of the job, syntax as a cron expression"
                      backupProfileName:
                        type: string
                        description: "Name of the backupProfile to be used"
                      backupProfile:
                        type: object
                        description: "backupProfile specification if backupProfileName is not specified"
                        x-kubernetes-preserve-unknown-fields: true
                        properties:
                          # ... (中略) ...
                      deleteBackupData:
                        type: boolean
                        default: false
                        description: "Whether to delete the backup data in case the MySQLBackup object created by the job is deleted"
                      enabled:
                        type: boolean
                        default: true
                        description: "Whether the schedule is enabled or not"
                      timeZone:
                        type: string
                        description: "Timezone for the backup schedule, example: 'America/New_York'"
```

主要フィールドをまとめると次のとおりである。

| フィールド | 必須 | 説明 |
|---|---|---|
| `name` | 必須 | スケジュールの名前 |
| `schedule` | 必須 | cron 形式の実行間隔 |
| `backupProfileName` | 任意 | 対象クラスタに登録済みの `backupProfiles` から名前で参照する |
| `backupProfile` | 任意 | `backupProfileName` を指定しない場合に、プロファイル定義をインラインで書く |
| `enabled` | 任意（既定値 `true`） | スケジュールを有効にするか |
| `deleteBackupData` | 任意（既定値 `false`） | フィールドは定義されているが、8.4.9-2.1.11 の実装では未使用である（後述） |
| `timeZone` | 任意 | cron 実行のタイムゾーン（例：`America/New_York`） |

### スケジュールバックアップを設定する

以下は、第15章のプロファイル `daily-dump-to-pvc` を毎日午前2時（UTC）に実行する例である。

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
  backupProfiles:
    - name: daily-dump-to-pvc
      dumpInstance:
        storage:
          persistentVolumeClaim:
            claimName: backup-pvc
  backupSchedules:
    - name: daily
      schedule: "0 2 * * *"
      backupProfileName: daily-dump-to-pvc
      enabled: true
```

適用後、operator は `<クラスタ名>-<スケジュール名>-cb` という名前の CronJob を生成する。

```bash
kubectl apply -f innodbcluster-with-schedule.yaml
kubectl get cronjob -l mysql.oracle.com/cluster=mycluster
```

```text
NAME                SCHEDULE     SUSPEND   ACTIVE   LAST SCHEDULE   AGE
mycluster-daily-cb  0 2 * * *    False     0         <none>         30s
```

`enabled: false` にすると、operator は同じ CronJob の `spec.suspend` を `true` に切り替える。
CronJob 自体を削除するのではなく一時停止する形になるため、再度 `enabled: true` に戻せば同じスケジュール名で再開できる。

```mermaid
flowchart LR
    A[InnoDBCluster\nspec.backupSchedules] --> B[CronJob\nmycluster-daily-cb]
    B -->|schedule どおりに起動| C[Job: create-backup-object]
    C --> D[MySQLBackup を1件作成]
    D --> E[Job: 実際のダンプを実行]
    E --> F[status を更新]
```

CronJob の実体は、`mysqlsh --pym mysqloperator backup --command create-backup-object` を実行するだけの軽量な Job である。
このコマンドが `MySQLBackup` オブジェクトを1件生成し、そのオブジェクトの作成をきっかけに第16章で見た `on_mysqlbackup_create` ハンドラが動き、実際のダンプを行う Job が作られる。
つまりスケジュールバックアップは、cron のタイミングでオンデマンドバックアップを自動的に起動する仕組みである。

### 実行結果を確認する

スケジュールによって生成された `MySQLBackup` は、`mysql.oracle.com/cluster` ラベルで一覧できる。

```bash
kubectl get mysqlbackup -l mysql.oracle.com/cluster=mycluster
```

```text
NAME                          CLUSTER     STATUS      OUTPUT                              AGE
mycluster-daily2412030200     mycluster   Completed   mycluster-daily2412030200            5h
```

`status.status` は第16章で見たとおり `Running` / `Completed` / `Error` のいずれかを取り、この例は正常に完了した状態を示す。

過去の実行結果は `MySQLBackup` オブジェクトとして蓄積され続けるため、古いものは運用ポリシーに応じて削除する。
`deleteBackupData` は名前からは削除時にバックアップデータ本体も消すように見えるが、第16章で見たとおり 8.4.9-2.1.11 の実装では未使用であり、`true` にしても保存先のデータは自動的には削除されない。
保存先が PVC であれば PVC 自体の削除、OCI/S3/Azure であればバケットまたはコンテナ側のライフサイクルポリシーなど、保存先の仕組みで運用者が削除する必要がある。

## トラブルシューティング

CronJob の `LAST SCHEDULE` が更新されない場合は、`concurrencyPolicy: Forbid` により前回の Job が実行中のまま次のスケジュールをスキップしている可能性がある。

```bash
kubectl get jobs -l mysql.oracle.com/cluster=mycluster --sort-by=.metadata.creationTimestamp
```

実行中の Job が滞留していないかを確認し、必要であれば失敗した Job を削除してから次回の実行を待つ。

## まとめ

- `backupSchedules` は cron 形式の `schedule` とプロファイル参照を組み合わせ、定期バックアップを CronJob として実現する。
- CronJob は直接ダンプを実行するのではなく、`MySQLBackup` オブジェクトを生成することでオンデマンドバックアップの仕組みを再利用する。
- `enabled` の切り替えは CronJob の一時停止に対応し、スケジュール自体の削除は不要である。

## 関連する章

- [第15章 バックアップの概念とプロファイル](15-backup-concepts.md)
- [第16章 オンデマンドバックアップ（MySQLBackup）](16-ondemand-backup.md)
- [第18章 リストア](18-restore.md)
