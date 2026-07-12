# 第10章 リスナー認証

> 本章で参照する公式リソース
>
> - [install/cluster-operator/040-Crd-kafka.yaml L100-L119](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/040-Crd-kafka.yaml#L100-L119)
> - [examples/security/tls-auth/kafka.yaml L47-L55](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/security/tls-auth/kafka.yaml#L47-L55)
> - [examples/security/scram-sha-512-auth/kafka.yaml L47-L55](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/security/scram-sha-512-auth/kafka.yaml#L47-L55)
> - [install/cluster-operator/044-Crd-kafkauser.yaml L87-L100](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/044-Crd-kafkauser.yaml#L87-L100)
> - [examples/user/kafka-user.yaml L1-L38](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/user/kafka-user.yaml#L1-L38)

## この章でできるようになること

- リスナーの `authentication.type`（`tls`、`scram-sha-512`、`custom`）を選定できる。
- mTLS と SCRAM の使い分けを説明できる。
- `KafkaUser` の認証設定との対応関係を理解できる。
- 認証付きリスナーへの接続を検証できる。

## 前提

[第9章 TLS と認証局](09-tls-certificates.md)で CA と TLS リスナーを理解していること。
本章は第3章のオープンクラスタ（`my-cluster`）を前提とする。

## authentication.type の種類

リスナーごとに認証方式を指定する。

[install/cluster-operator/040-Crd-kafka.yaml L100-L119](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/040-Crd-kafka.yaml#L100-L119)は次のとおりである。

```yaml
                        authentication:
                          type: object
                          properties:
                            listenerConfig:
                              x-kubernetes-preserve-unknown-fields: true
                              type: object
                              description: Configuration to be used for a specific listener. All values are prefixed with `listener.name.<listener_name>`.
                            sasl:
                              type: boolean
                              description: Enable or disable SASL on this listener.
                            type:
                              type: string
                              enum:
                              - tls
                              - scram-sha-512
                              - custom
                              description: Authentication type. `scram-sha-512` type uses SASL SCRAM-SHA-512 Authentication. `tls` type uses TLS Client Authentication. `tls` type is supported only on TLS listeners. `custom` type allows for any authentication type to be used.
                          required:
                          - type
                          description: Authentication configuration for this listener.
```

| type | 説明 | リスナー要件 |
|---|---|---|
| `tls` | クライアント証明書による mTLS | `tls: true` |
| `scram-sha-512` | SASL SCRAM-SHA-512 | 暗号化または非暗号化のいずれのリスナーでも可（TLS 推奨） |
| `custom` | 任意の認証（OAuth など）を `listenerConfig` で指定 | 設定による |

OAuth 2.0 は `custom` と `listenerConfig` に加え `sasl: true` で構成する（[第11章](11-authorization.md)の Keycloak 例を参照）。

## mTLS（type: tls）

[examples/security/tls-auth/kafka.yaml L47-L55](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/security/tls-auth/kafka.yaml#L47-L55)は TLS リスナーに mTLS を設定する。

```yaml
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
    authorization:
      type: simple
```

クライアントは cluster CA 証明書でサーバーを検証し、ユーザー証明書で自身を証明する。
`KafkaUser` の `authentication.type: tls` と組み合わせ、User Operator が証明書を Secret に生成する。

## SCRAM（type: scram-sha-512）

[examples/security/scram-sha-512-auth/kafka.yaml L47-L55](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/security/scram-sha-512-auth/kafka.yaml#L47-L55)は SCRAM 認証を有効にする。

```yaml
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: scram-sha-512
    authorization:
      type: simple
```

`KafkaUser` 側は `authentication.type: scram-sha-512` とし、User Operator がパスワードを Secret に格納する。
Strimzi は暗号化または非暗号化の両接続で SCRAM を許可する。

[install/cluster-operator/044-Crd-kafkauser.yaml L87-L100](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/install/cluster-operator/044-Crd-kafkauser.yaml#L87-L100)は次のとおりである。

```yaml
                  type:
                    type: string
                    enum:
                    - tls
                    - tls-external
                    - scram-sha-512
                    description: Authentication type.
                  validityDays:
                    type: integer
                    minimum: 1
                    description: "Number of days for which the user certificate is valid. Both this property and `renewalDays` must be configured together.The value must be greater than 0 and greater than `renewalDays`.If not configured, the Clients CA configuration is used."
                required:
                - type
                description: "Authentication mechanism enabled for this Kafka user. The supported authentication mechanisms are `scram-sha-512`, `tls`, and `tls-external`. \n\n* `scram-sha-512` generates a secret with SASL SCRAM-SHA-512 credentials.\n* `tls` generates a secret with user certificate for mutual TLS authentication.\n* `tls-external` does not generate a user certificate.   But prepares the user for using mutual TLS authentication using a user certificate generated outside the User Operator.\n  ACLs and quotas set for this user are configured in the `CN=<username>` format.\n\nAuthentication is optional. If authentication is not configured, no credentials are generated. ACLs and quotas set for the user are configured in the `<username>` format suitable for SASL authentication."
```

## mTLS と SCRAM の使い分け

mTLS は証明書ローテーションの自動化と強い本人確認に向く。
SCRAM はユーザー名とパスワードベースで、クライアントライブラリの互換性が広い。
いずれも TLS リスナー上で使うのが一般的である（SCRAM の平文送信は避ける）。

```mermaid
flowchart LR
  subgraph mTLS [mTLS]
    U1[KafkaUser tls] --> S1[Secret 証明書]
    S1 --> C1[クライアント]
  end
  subgraph SCRAM [SCRAM]
    U2[KafkaUser scram] --> S2[Secret パスワード]
    S2 --> C2[クライアント]
  end
```

## 動作確認

第3章のオープンクラスタから、本章の手順だけで mTLS 接続を再現する。
以下は動作確認用の手順例である。

まず TLS リスナー認証と simple 認可を有効化する。

```bash
kubectl patch kafka my-cluster -n kafka --type=merge -p '
{
  "spec": {
    "kafka": {
      "authorization": {"type": "simple"},
      "listeners": [
        {"name": "plain", "port": 9092, "type": "internal", "tls": false},
        {"name": "tls", "port": 9093, "type": "internal", "tls": true, "authentication": {"type": "tls"}}
      ]
    }
  }
}'
```

期待される出力の例は次のとおりである。

```text
kafka.kafka.strimzi.io/my-cluster patched
```

patch 後は `observedGeneration` が `generation` に追いつくのを待ってから Ready を確認する。

```bash
GEN=$(kubectl get kafka my-cluster -n kafka -o jsonpath='{.metadata.generation}')
kubectl wait kafka/my-cluster -n kafka \
  --for=jsonpath="{.status.observedGeneration}=${GEN}" --timeout=600s
kubectl wait kafka/my-cluster -n kafka --for=condition=Ready --timeout=600s
```

期待される出力の例は次のとおりである。

```text
kafka.kafka.strimzi.io/my-cluster condition met
kafka.kafka.strimzi.io/my-cluster condition met
```

[examples/user/kafka-user.yaml L1-L38](https://github.com/strimzi/strimzi-kafka-operator/blob/1.1.0/examples/user/kafka-user.yaml#L1-L38)は次のとおりである。

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaUser
metadata:
  name: my-user
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      # Example consumer Acls for topic my-topic using consumer group my-group
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operations:
          - Describe
          - Read
        host: "*"
      - resource:
          type: group
          name: my-group
          patternType: literal
        operations:
          - Read
        host: "*"
      # Example Producer Acls for topic my-topic
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operations:
          - Create
          - Describe
          - Write
        host: "*"
```

```bash
kubectl apply -f kafka-user.yaml -n kafka
kubectl wait kafkauser/my-user -n kafka --for=condition=Ready --timeout=120s
```

期待される出力の例は次のとおりである。

```text
kafkauser.kafka.strimzi.io/my-user created
kafkauser.kafka.strimzi.io/my-user condition met
```

```bash
kubectl get kafkauser my-user -n kafka
```

期待される出力の例は次のとおりである。

```text
NAME      CLUSTER      AUTHENTICATION   AUTHORIZATION   READY
my-user   my-cluster   tls              simple          True
```

TLS 認証付きクラスタで、ユーザー Secret と cluster CA を使って consumer を起動する。
PEM 内容は改行を `\n` に置換した単一行形式で書く。

```bash
kubectl run kafka-consumer-auth -ti --restart=Never -n kafka \
  --image=quay.io/strimzi/kafka:1.1.0-kafka-4.3.0 \
  --overrides='{"spec":{"volumes":[{"name":"user","secret":{"secretName":"my-user"}},{"name":"cluster-ca","secret":{"secretName":"my-cluster-cluster-ca-cert"}}],"containers":[{"name":"kafka","image":"quay.io/strimzi/kafka:1.1.0-kafka-4.3.0","command":["/bin/bash","-c"],"args":["format_pem() { awk \"NF {sub(/\\\\r/, \\\"\\\"); printf \\\"%s\\\\\\\\n\\\",\\$0;}\" \"$1\" | sed \"s/\\\\\\\\n$//\"; }; TRUST=$(format_pem /mnt/cluster-ca/ca.crt); CERT=$(format_pem /mnt/user/user.crt); KEY=$(format_pem /mnt/user/user.key); cat > /tmp/client.properties <<EOF\nsecurity.protocol=SSL\nssl.truststore.type=PEM\nssl.truststore.certificates=${TRUST}\nssl.keystore.type=PEM\nssl.keystore.certificate.chain=${CERT}\nssl.keystore.key=${KEY}\nEOF\nbin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9093 --topic my-topic --from-beginning --group my-group --consumer.config /tmp/client.properties"],"volumeMounts":[{"name":"user","mountPath":"/mnt/user"},{"name":"cluster-ca","mountPath":"/mnt/cluster-ca"}]}]}}'
```

Pod 内で生成される `client.properties` には次のキーが含まれる（PEM は単一行形式）。

```properties
security.protocol=SSL
ssl.truststore.type=PEM
ssl.truststore.certificates=-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----
ssl.keystore.type=PEM
ssl.keystore.certificate.chain=-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----
ssl.keystore.key=-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----
```

ブローカー証明書の検証には `<cluster>-cluster-ca-cert` の `ca.crt` を使う。
`my-user` Secret の `ca.crt` は clients CA であり、サーバー信頼の根拠にはならない。

認証が通ればトピックのメッセージが表示される。
認証失敗時は `SSL handshake failed` や `Authentication failed` などのエラーがログに出る。

期待される出力の例は次のとおりである。

```text
hello-from-quickstart
```

第3章で producer が送信済みのメッセージが表示される。
未送信の場合は第3章の producer 手順を先に実行する。

## まとめ

リスナーの `authentication.type` で mTLS または SCRAM を選ぶ。
`KafkaUser` の認証 type はリスナーと対応させる（`tls-external` はリスナー側 `type: tls` を使い、User Operator は証明書を生成しない）。
OAuth は `custom` type、`sasl: true`、`listenerConfig` で拡張する。
本章で Kafka に加えた認証設定は例示であり、以降の章は第3章のオープンクラスタを前提に読む。

## 関連する章

- [第9章 TLS と認証局](09-tls-certificates.md)
- [第11章 認可と ACL](11-authorization.md)
