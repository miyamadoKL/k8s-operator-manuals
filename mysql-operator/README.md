# MySQL Operator 使用方法マニュアル

Oracle MySQL Operator（[mysql/mysql-operator](https://github.com/mysql/mysql-operator)）の導入から運用までを日本語で解説するドキュメントである。

- **対象バージョン**：8.4.9-2.1.11（引用はすべて [`8.4.9-2.1.11` タグ](https://github.com/mysql/mysql-operator/tree/8.4.9-2.1.11)に固定）
- **対象 operator リポジトリ**：[mysql/mysql-operator](https://github.com/mysql/mysql-operator)
- **想定読者**：Kubernetes を触った経験があるインフラエンジニアやアプリケーション開発者
- **読み方**：第0部から順に読めば、最短でクラスタを起動し、リファレンスと運用まで到達できる。

公式リソース（CRD 定義、Helm values、サンプル）の引用はバージョン固定の GitHub リンクとコードブロックの2点セットで示す。
本書は community 版を主軸に解説し、enterprise 版は差分を必要な箇所で注記する。

## 第0部　導入

1. [MySQL Operator の概要とアーキテクチャ](part00-introduction/01-overview.md)
2. [Operator のインストール](part00-introduction/02-install-operator.md)
3. [クイックスタート](part00-introduction/03-quickstart.md)

## 第1部　InnoDBCluster リファレンス（基本）

4. [InnoDBCluster リソースの全体像](part01-innodbcluster-basics/04-innodbcluster-resource.md)
5. [認証情報と Secret](part01-innodbcluster-basics/05-credentials-secret.md)
6. [サーバー構成（インスタンス数、バージョン、Edition）](part01-innodbcluster-basics/06-server-config.md)
7. [ストレージと PersistentVolumeClaim](part01-innodbcluster-basics/07-storage-pvc.md)
8. [MySQL 設定（mycnf）](part01-innodbcluster-basics/08-mycnf.md)
9. [Pod 仕様のカスタマイズ（podSpec）](part01-innodbcluster-basics/09-podspec.md)

## 第2部　ネットワークと接続

10. [Service と接続エンドポイント](part02-networking/10-service-connection.md)
11. [MySQL Router](part02-networking/11-mysql-router.md)
12. [TLS と証明書](part02-networking/12-tls.md)

## 第3部　データ初期化とセキュリティ

13. [データの初期化（initDB）](part03-initdb-security/13-initdb.md)
14. [Keyring と保存時暗号化](part03-initdb-security/14-keyring-encryption.md)

## 第4部　バックアップとリストア

15. [バックアップの概念とプロファイル](part04-backup-restore/15-backup-concepts.md)
16. [オンデマンドバックアップ（MySQLBackup）](part04-backup-restore/16-ondemand-backup.md)
17. [スケジュールバックアップ](part04-backup-restore/17-scheduled-backup.md)
18. [リストア](part04-backup-restore/18-restore.md)

## 第5部　運用

19. [スケーリングとアップグレード](part05-operations/19-scaling-upgrade.md)
20. [Read Replica](part05-operations/20-read-replicas.md)
21. [メトリクスとログ](part05-operations/21-metrics-logs.md)
22. [トラブルシューティングと診断](part05-operations/22-troubleshooting.md)

---

> 本書は Oracle MySQL Operator `8.4.9-2.1.11` を対象に執筆した全22章のマニュアルである。
> 引用はすべて同タグに固定している。
