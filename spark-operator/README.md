# Spark Operator 使用方法マニュアル

本書は、Spark Operator（[kubeflow/spark-operator](https://github.com/kubeflow/spark-operator)）の導入から運用までを日本語で解説するドキュメントである。

- **対象バージョン**：2.5.1（引用はすべて [`v2.5.1` タグ](https://github.com/kubeflow/spark-operator/tree/v2.5.1)に固定）
- **対象 operator リポジトリ**：[kubeflow/spark-operator](https://github.com/kubeflow/spark-operator)
- **想定読者**：Kubernetes を触った経験があるインフラ・アプリ開発者
- **読み方**：第0部から順に読み進めると、導入から運用まで段階的に理解できる

公式リソースからの引用にはバージョン固定の GitHub リンクを添え、自作のマニフェスト例にはその旨を本文で明示する。

## 第0部　導入

1. [Spark Operator とは](part00-introduction/01-what-is-spark-operator.md)
2. [インストール](part00-introduction/02-installation.md)
3. [クイックスタート](part00-introduction/03-quickstart.md)

## 第1部　SparkApplication の基本

4. [SparkApplication の基本構造](part01-sparkapplication-basics/04-sparkapplication-spec.md)
5. [driver と executor の設定](part01-sparkapplication-basics/05-driver-executor.md)
6. [依存関係とアプリケーションの配布](part01-sparkapplication-basics/06-dependencies.md)
7. [ライフサイクルと再実行](part01-sparkapplication-basics/07-lifecycle.md)

## 第2部　SparkApplication の応用

8. [Volume と Pod テンプレート](part02-sparkapplication-advanced/08-volumes-podtemplate.md)
9. [動的リソース割り当てとバッチスケジューラ](part02-sparkapplication-advanced/09-dynamic-allocation-schedulers.md)
10. [監視とメトリクス](part02-sparkapplication-advanced/10-monitoring.md)
11. [セキュリティと認証](part02-sparkapplication-advanced/11-security.md)

## 第3部　その他の CRD

12. [ScheduledSparkApplication](part03-other-crds/12-scheduledsparkapplication.md)
13. [SparkConnect](part03-other-crds/13-sparkconnect.md)

## 第4部　Helm chart と運用

14. [Helm values リファレンス](part04-operations/14-helm-values.md)
15. [Webhook とデフォルト注入](part04-operations/15-webhook.md)
16. [運用とトラブルシューティング](part04-operations/16-operations-troubleshooting.md)

---

> 全章執筆中である。
> 引用は v2.5.1 タグに固定する。
