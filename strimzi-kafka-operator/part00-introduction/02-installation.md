# 第2章 インストール

> 本章で参照する公式リソース
>
> - [install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml L1-L46](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml#L1-L46)
> - [install/cluster-operator/010-ServiceAccount-strimzi-cluster-operator.yaml L1-L6](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/010-ServiceAccount-strimzi-cluster-operator.yaml#L1-L6)
> - [release.version L1-L1](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/release.version#L1-L1)
> - [Makefile L97-L101](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/Makefile#L97-L101)
> - [install/cluster-operator/021-ClusterRole-strimzi-cluster-operator-role.yaml L1-L21](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/021-ClusterRole-strimzi-cluster-operator-role.yaml#L1-L21)
> - [install/cluster-operator/020-ClusterRole-strimzi-cluster-operator-role.yaml L38-L61](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/020-ClusterRole-strimzi-cluster-operator-role.yaml#L38-L61)
> - [helm-charts/helm3/strimzi-kafka-operator/values.yaml L3-L8](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/values.yaml#L3-L8)
> - [helm-charts/helm3/strimzi-kafka-operator/Chart.yaml L1-L6](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/Chart.yaml#L1-L6)

## この章でできるようになること

- Cluster Operator を kubectl apply または Helm でインストールできる。
- 監視対象 Namespace の設定方法（`watchNamespaces` / `watchAnyNamespace`）を理解できる。
- RBAC と ServiceAccount の役割を説明できる。
- インストール後の動作確認コマンドを実行できる。

## 前提

kubectl がクラスタに接続でき、Cluster Admin 相当の権限で CRD と ClusterRole を作成できることを前提とする。
Helm を使う手順では Helm 3 が利用可能であることも前提とする。
本章は Kafka クラスタの有無に依存せず、Cluster Operator の導入のみを扱う。

## インストール方式の選択

Cluster Operator のインストールには次の 2 通りがある。

1. `install/cluster-operator/` 配下の YAML を `kubectl apply` する。
2. `helm-charts/helm3/strimzi-kafka-operator` の Helm chart を使う。

どちらも同じ Operator イメージ（`quay.io/strimzi/operator:1.1.0`）をデプロイする。
本番環境では Namespace の監視範囲やリソース制限を Helm values で調整するケースが多い。

## 手順 A: kubectl apply によるインストール

Operator 用の Namespace を作成し、リリース成果物を取得する。
RoleBinding と ClusterRoleBinding の `subjects` に含まれる Namespace は、インストール先に合わせて置換する必要がある。

```bash
kubectl create namespace strimzi
```

期待される出力の例は次のとおりである。

```text
namespace/strimzi created
```

```bash
curl -Lo strimzi-cluster-operator-1.1.0.yaml \
  https://github.com/strimzi/strimzi-kafka-operator/releases/download/1.1.0/strimzi-cluster-operator-1.1.0.yaml
sed -i 's/namespace: myproject/namespace: strimzi/g' strimzi-cluster-operator-1.1.0.yaml
kubectl apply -f strimzi-cluster-operator-1.1.0.yaml -n strimzi
```

期待される出力の例は次のとおりである。
リリース bundle には 27 リソースが含まれる。
適用順は bundle の文書順や kubectl の visitor 順により環境で異なる。

```text
configmap/strimzi-cluster-operator created
clusterrole.rbac.authorization.k8s.io/strimzi-cluster-operator-namespaced created
customresourcedefinition.apiextensions.k8s.io/kafkaconnectors.kafka.strimzi.io created
# ...（以下略。適用順・件数は環境により異なる）
serviceaccount/strimzi-cluster-operator created
deployment.apps/strimzi-cluster-operator created
```

`strimzi-cluster-operator-1.1.0.yaml` は `install/cluster-operator/` 配下のマニフェストを束ねたリリース成果物である。
`sed` を省略すると ServiceAccount は `strimzi` に作成されても、Binding の subject が `myproject` のまま残り、Operator に権限が結び付かない。
CRD、RBAC、ConfigMap、Deployment がまとめて適用される。

動作確認は次のとおりである。

```bash
kubectl get deployment strimzi-cluster-operator -n strimzi
```

期待される出力の例は次のとおりである。

```text
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
strimzi-cluster-operator   1/1     1            1           2m
```

CRD が登録されたことを確認する。

```bash
kubectl get crd | grep strimzi
```

期待される出力には、少なくとも次の CRD 名が `NAME` 列と `CREATED AT` 列付きで含まれる。
一覧順と時刻の値は環境により異なる。

```text
NAME                                    CREATED AT
kafkas.kafka.strimzi.io                 <timestamp>
kafkanodepools.kafka.strimzi.io         <timestamp>
# ...（以下略。順序や時刻列は環境により異なる）
strimzipodsets.core.strimzi.io          <timestamp>
```

## 手順 B: Helm によるインストール

[helm-charts/helm3/strimzi-kafka-operator/Chart.yaml L1-L6](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/Chart.yaml#L1-L6)は次のとおりである。

```yaml
apiVersion: v2
appVersion: "0.1.0"
description: "Strimzi: Apache Kafka running on Kubernetes"
name: strimzi-kafka-operator
version: 0.1.1
icon: https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/main/documentation/logo/strimzi_logo.png
```

Helm リポジトリを追加して chart をインストールする。

```bash
helm repo add strimzi https://strimzi.io/charts/
```

期待される出力の例は次のとおりである。

```text
"strimzi" has been added to your repositories
```

```bash
helm repo update
```

期待される出力の例は次のとおりである。
Helm のバージョンにより前後の行の文言は異なる。

```text
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "strimzi" chart repository
# ...（以下略。順序や時刻列は環境により異なる）
```

```bash
kubectl create namespace strimzi
helm install strimzi-kafka-operator strimzi/strimzi-kafka-operator \
  --namespace strimzi \
  --version 1.1.0
```

期待される出力の例は次のとおりである。
`LAST DEPLOYED` の日時は環境により異なる。

```text
namespace/strimzi created
NAME: strimzi-kafka-operator
STATUS: deployed
# ...（以下略。順序や時刻列は環境により異なる）
NOTES:
Thank you for installing strimzi-kafka-operator-1.1.0
```

chart の `version` は [release.version L1-L1](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/release.version#L1-L1)と [Makefile L97-L101](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/Makefile#L97-L101)の `helm package --version` で同じ値（1.1.0）に揃えられる。

```text
1.1.0
```

```makefile
helm_pkg: dashboard_install
	# Copying unarchived Helm Chart to release directory
	mkdir -p strimzi-$(RELEASE_VERSION)/helm3-charts/
	helm package --version $(CHART_SEMANTIC_RELEASE_VERSION) --app-version $(CHART_SEMANTIC_RELEASE_VERSION) --destination ./ ./packaging/helm-charts/helm3/strimzi-kafka-operator/
	$(CP) strimzi-kafka-operator-$(CHART_SEMANTIC_RELEASE_VERSION).tgz strimzi-kafka-operator-helm-3-chart-$(CHART_SEMANTIC_RELEASE_VERSION).tgz
```

`helm search repo strimzi/strimzi-kafka-operator` で利用可能な chart バージョンを確認できる。

動作確認は手順 A と同じコマンドで行う。

## 監視対象 Namespace の設定

Cluster Operator は環境変数 `STRIMZI_NAMESPACE` で監視対象を決める。
kubectl apply 方式では、Deployment の `STRIMZI_NAMESPACE` が Operator 自身の Namespace を指す。

[install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml L41-L47](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml#L41-L47)は次のとおりである。

```yaml
          env:
            - name: STRIMZI_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: STRIMZI_FULL_RECONCILIATION_INTERVAL_MS
              value: "120000"
```

Helm では [helm-charts/helm3/strimzi-kafka-operator/values.yaml L3-L8](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/helm-charts/helm3/strimzi-kafka-operator/values.yaml#L3-L8)で制御する。

```yaml
# Default replicas for the cluster operator
replicas: 1

# Contains `.Release.Namespace` by default
watchNamespaces: []
watchAnyNamespace: false
```

`watchNamespaces` が空のとき、Operator は chart をインストールした Namespace のみを監視する。
複数の Namespace で Kafka を運用する場合は、監視対象のリストを values で指定する。
Helm の helper は常に `.Release.Namespace` も監視対象に加える。

以下は `kafka-prod` と `kafka-dev` を追加監視する values の例である。

```yaml
watchNamespaces:
  - kafka-prod
  - kafka-dev
watchAnyNamespace: false
```

上記を `strimzi` Namespace にインストールすると、実際の監視対象は `kafka-dev`、`kafka-prod`、`strimzi` の 3 つになる。

`watchAnyNamespace: true` にすると、クラスタ内のすべての Namespace を監視する。
ClusterRole による広い権限が必要になるため、本番では Namespace を限定する構成を検討する。

## RBAC と ServiceAccount

Cluster Operator は専用の ServiceAccount で動作する。

[install/cluster-operator/010-ServiceAccount-strimzi-cluster-operator.yaml L1-L6](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/010-ServiceAccount-strimzi-cluster-operator.yaml#L1-L6)は次のとおりである。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: strimzi-cluster-operator
  labels:
    app: strimzi
```

ClusterRole はクラスタスコープの権限を定義する。
Pod、PVC、Secret の作成権限は [install/cluster-operator/020-ClusterRole-strimzi-cluster-operator-role.yaml L38-L61](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/020-ClusterRole-strimzi-cluster-operator-role.yaml#L38-L61)にある `strimzi-cluster-operator-namespaced` に含まれる。

```yaml
  - apiGroups:
      - ""
    resources:
      # The cluster operator needs to access and delete pods, this is to allow it to monitor pod health and coordinate rolling updates
      - pods
      # The cluster operator needs to access and manage service accounts to grant Strimzi components cluster permissions
      - serviceaccounts
      # The cluster operator needs to access and manage config maps for Strimzi components configuration
      - configmaps
      # The cluster operator needs to access and manage services and endpoints to expose Strimzi components to network traffic
      - services
      - endpoints
      # The cluster operator needs to access and manage secrets to handle credentials
      - secrets
      # The cluster operator needs to access and manage persistent volume claims to bind them to Strimzi components for persistent data
      - persistentvolumeclaims
    verbs:
      - get
      - list
      - watch
      - create
      - delete
      - patch
      - update
```

[install/cluster-operator/021-ClusterRole-strimzi-cluster-operator-role.yaml L1-L21](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/021-ClusterRole-strimzi-cluster-operator-role.yaml#L1-L21)は、グローバル権限を定義する別の ClusterRole である。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: strimzi-cluster-operator-global
  labels:
    app: strimzi
rules:
  - apiGroups:
      - "rbac.authorization.k8s.io"
    resources:
      # The cluster operator needs to create and manage cluster role bindings in the case of an install where a user
      # has specified they want their cluster role bindings generated
      - clusterrolebindings
    verbs:
      - get
      - list
      - watch
      - create
      - delete
      - patch
      - update
```

`020` から `023` 番台のマニフェストは、ClusterRole を RoleBinding または ClusterRoleBinding で ServiceAccount に結び付ける。
Kafka をデプロイする各監視対象 Namespace には RoleBinding を追加する。

## トラブルシューティング

Deployment が `READY 0/1` のままの場合、Pod のイベントを確認する。

```bash
kubectl describe pod -l name=strimzi-cluster-operator -n strimzi
```

期待される出力の `Events` 節に Warning がないことを確認する。

```text
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  2m    default-scheduler  Successfully assigned strimzi/strimzi-cluster-operator-...
  Normal  Pulled     2m    kubelet            Container image "quay.io/strimzi/operator:1.1.0" already present
```

CRD の作成権限がない場合、`kubectl apply` が `Forbidden` で失敗する。
Cluster Admin に CRD インストールを依頼するか、クラスタ管理者権限を持つアカウントで再実行する。

## まとめ

Cluster Operator は kubectl apply または Helm のどちらでもインストールできる。
監視対象 Namespace は `STRIMZI_NAMESPACE` または Helm values の `watchNamespaces` で制御する。
インストール後は Deployment と CRD の存在を確認して完了とする。

## 関連する章

- [第1章 Strimzi Kafka Operator とは](01-what-is-strimzi.md)
- [第3章 クイックスタート](03-quickstart.md)
