# Strimzi Kafka Operator 使用方法マニュアル

本書は、Strimzi Kafka Operator（[strimzi/strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)）の導入から運用までを日本語で解説するドキュメントである。

- **対象バージョン**：1.1.0（引用はすべて [`1.1.0` タグ](https://github.com/strimzi/strimzi-kafka-operator/tree/1.1.0)に固定）
- **対象 operator リポジトリ**：[strimzi/strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)
- **想定読者**：Kubernetes を触った経験があるインフラ・アプリ開発者
- **読み方**：第0部から順に読み進めると、導入から運用まで段階的に理解できる

公式リソースからの引用にはバージョン固定の GitHub リンクを添え、自作のマニフェスト例にはその旨を本文で明示する。
本書は KRaft モード（ZooKeeper を使わない構成）を前提とする。

## 第0部　導入

1. [Strimzi Kafka Operator とは](part00-introduction/01-what-is-strimzi.md)
2. [インストール](part00-introduction/02-installation.md)
3. [クイックスタート](part00-introduction/03-quickstart.md)

## 第1部　Kafka クラスタの構築

4. [KafkaNodePool とノードロール](part01-kafka-cluster/04-kafkanodepool.md)
5. [Kafka カスタムリソースの基本構造](part01-kafka-cluster/05-kafka-resource.md)
6. [ストレージ設定](part01-kafka-cluster/06-storage.md)
7. [リスナーと外部アクセス](part01-kafka-cluster/07-listeners.md)
8. [リソース、スケジューリング、Pod テンプレート](part01-kafka-cluster/08-scheduling-template.md)

## 第2部　セキュリティ

9. [TLS と認証局](part02-security/09-tls-certificates.md)
10. [リスナー認証](part02-security/10-authentication.md)
11. [認可と ACL](part02-security/11-authorization.md)

## 第3部　トピックとユーザー

12. [KafkaTopic の管理](part03-topics-users/12-kafkatopic.md)
13. [KafkaUser の管理](part03-topics-users/13-kafkauser.md)

## 第4部　Kafka Connect

14. [KafkaConnect の構築](part04-connect/14-kafkaconnect.md)
15. [コネクタープラグインのビルド](part04-connect/15-connect-build.md)
16. [KafkaConnector によるコネクター管理](part04-connect/16-kafkaconnector.md)

## 第5部　レプリケーションとブリッジ

17. [KafkaMirrorMaker2 によるクラスタ間レプリケーション](part05-replication-bridge/17-mirrormaker2.md)
18. [KafkaBridge による HTTP アクセス](part05-replication-bridge/18-kafkabridge.md)

## 第6部　リバランスとスケーリング

19. [Cruise Control の有効化](part06-cruise-control/19-cruise-control.md)
20. [KafkaRebalance によるリバランス](part06-cruise-control/20-kafkarebalance.md)

## 第7部　運用

21. [Helm values リファレンス](part07-operations/21-helm-values.md)
22. [監視とメトリクス](part07-operations/22-monitoring-metrics.md)
23. [スケーリングとローリング更新](part07-operations/23-scaling-rolling-update.md)
24. [トラブルシューティングと運用](part07-operations/24-troubleshooting.md)

---

> 全24章の初版が完成している。
> 引用は 1.1.0 タグに固定する。
