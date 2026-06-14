# 04 — Kafka (Strimzi) + ELK (ECK) — giám sát log production

Kiến trúc:
```
Pod → Filebeat (DaemonSet, TLS) → Kafka cluster (Strimzi, TLS+SASL+ACL)
    → Logstash (persistent queue) → Elasticsearch (ECK, node roles, TLS+auth) → Kibana
```

Khác bản lab: **mọi kết nối mã hóa + xác thực**, Kafka quản lý bằng **Strimzi operator** (chuẩn production trên K8s), Elasticsearch **tách vai trò node**, có **snapshot ra S3** và **ILM** quản lý vòng đời log.

---

## 1. Elasticsearch cluster + Kibana (ECK)

### 1.1 Operator ECK (bản mới nhất)
```bash
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
kubectl -n elastic-system get pod
```
> ECK 3.4 yêu cầu Kubernetes ≥ 1.31. ECK bật **TLS + xác thực mặc định** — giữ nguyên (không tắt như bản lab).

### 1.2 ES với vai trò node tách bạch — `es-cluster.yaml`
```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata: { name: logging-es, namespace: logging }
spec:
  version: 9.4.2
  nodeSets:
    - name: master
      count: 3
      config: { node.roles: ["master"] }
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources: { requests: { memory: 2Gi, cpu: "1" }, limits: { memory: 2Gi } }
      volumeClaimTemplates:
        - metadata: { name: elasticsearch-data }
          spec: { accessModes: [ReadWriteOnce], resources: { requests: { storage: 10Gi } }, storageClassName: <ten-sc> }
    - name: data
      count: 3
      config: { node.roles: ["data", "ingest"] }
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources: { requests: { memory: 4Gi, cpu: "2" }, limits: { memory: 4Gi } }
      volumeClaimTemplates:
        - metadata: { name: elasticsearch-data }
          spec: { accessModes: [ReadWriteOnce], resources: { requests: { storage: 100Gi } }, storageClassName: <ten-sc> }
```
```bash
kubectl apply -f es-cluster.yaml
kubectl -n logging get elasticsearch    # HEALTH=green
```

### 1.3 Kibana qua Ingress + TLS — `kibana.yaml`
```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata: { name: logging-kibana, namespace: logging }
spec:
  version: 9.4.2
  count: 2
  elasticsearchRef: { name: logging-es }
  http:
    tls: { selfSignedCertificate: { disabled: true } }   # TLS do ingress đảm nhиệm
```
Tạo Ingress `kibana.example.com` (cert-manager) trỏ vào service `logging-kibana-kb-http`. Mật khẩu user `elastic` lấy từ secret `logging-es-es-elastic-user`; production nên cấu hình **OIDC/SSO** cho Kibana.

### 1.4 Snapshot ra S3 (DR) + ILM
```bash
# Đăng ký repository snapshot S3 (chạy qua curl tới ES, có auth + CA)
PW=$(kubectl -n logging get secret logging-es-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
kubectl -n logging exec logging-es-es-master-0 -- sh -c \
 "curl -s -k -u elastic:$PW -X PUT 'https://localhost:9200/_snapshot/s3_backup' \
  -H 'Content-Type: application/json' -d '{\"type\":\"s3\",\"settings\":{\"bucket\":\"es-snapshots\",\"endpoint\":\"s3.example.com\"}}'"
```
Tạo **ILM policy** xoay vòng (hot→warm→delete) để khống chế dung lượng log; gắn vào index template `k8s-logs-*`. Đây là yêu cầu bắt buộc production để log không phình vô hạn.

---

## 2. Kafka cluster qua Strimzi (TLS + SASL + ACL)

### 2.1 Cài operator
```bash
helm install strimzi strimzi/strimzi-kafka-operator -n logging
kubectl -n logging get pod -l name=strimzi-cluster-operator
```

### 2.2 Cụm Kafka 3 broker (KRaft) — `kafka.yaml`
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: broker
  namespace: logging
  labels: { strimzi.io/cluster: logging-kafka }
spec:
  replicas: 3
  roles: [controller, broker]
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 50Gi
        class: <ten-sc>
        deleteClaim: false
---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: logging-kafka
  namespace: logging
  annotations: { strimzi.io/node-pools: enabled, strimzi.io/kraft: enabled }
spec:
  kafka:
    version: 3.8.0
    replicas: 3
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication: { type: tls }      # mTLS
    authorization: { type: simple }          # bật ACL
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
  entityOperator: { topicOperator: {}, userOperator: {} }
```
```bash
kubectl apply -f kafka.yaml
kubectl -n logging get kafka,strimzipodset
```

### 2.3 Topic + user (ACL) cho log — `kafka-topic-user.yaml`
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata: { name: k8s-logs, namespace: logging, labels: { strimzi.io/cluster: logging-kafka } }
spec: { partitions: 6, replicas: 3, config: { retention.ms: 604800000 } }
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata: { name: filebeat, namespace: logging, labels: { strimzi.io/cluster: logging-kafka } }
spec:
  authentication: { type: tls }
  authorization:
    type: simple
    acls:
      - resource: { type: topic, name: k8s-logs }
        operations: [Write, Describe]
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata: { name: logstash, namespace: logging, labels: { strimzi.io/cluster: logging-kafka } }
spec:
  authentication: { type: tls }
  authorization:
    type: simple
    acls:
      - resource: { type: topic, name: k8s-logs }
        operations: [Read, Describe]
      - resource: { type: group, name: logstash }
        operations: [Read]
```
Strimzi tự tạo secret chứng chỉ client cho mỗi KafkaUser (vd `filebeat`, `logstash`) — dùng để mount vào Filebeat/Logstash.

---

## 3. Logstash (persistent queue, đọc Kafka mTLS → ghi ES)

`logstash.yaml` (ConfigMap + Deployment 2 node, mount chứng chỉ Kafka user `logstash` và CA của ES):
```
input {
  kafka {
    bootstrap_servers => "logging-kafka-kafka-bootstrap.logging.svc:9093"
    topics => ["k8s-logs"]
    group_id => "logstash"
    codec => json
    security_protocol => "SSL"
    ssl_truststore_location => "/certs/kafka/ca.p12"
    ssl_keystore_location => "/certs/kafka/user.p12"
  }
}
output {
  elasticsearch {
    hosts => ["https://logging-es-es-http.logging.svc:9200"]
    user => "elastic"
    password => "${ES_PASSWORD}"
    ssl_certificate_authorities => "/certs/es/ca.crt"
    ilm_enabled => true
    ilm_rollover_alias => "k8s-logs"
    ilm_policy => "k8s-logs-policy"
  }
}
```
Bật **persistent queue** (`queue.type: persisted`) trong `logstash.yml` để không mất sự kiện khi Logstash restart. Đặt resources requests=limits, 2 replica.

---

## 4. Filebeat (DaemonSet) → Kafka qua TLS

`filebeat.yml` (điểm khác bản lab: output Kafka có TLS + client cert của KafkaUser `filebeat`):
```yaml
filebeat.inputs:
  - type: container
    paths: [ /var/log/containers/*.log ]
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers: [ { logs_path: { logs_path: "/var/log/containers/" } } ]
output.kafka:
  hosts: ["logging-kafka-kafka-bootstrap.logging.svc:9093"]
  topic: "k8s-logs"
  ssl.certificate_authorities: ["/certs/ca.crt"]
  ssl.certificate: "/certs/user.crt"
  ssl.key: "/certs/user.key"
  required_acks: 1
  codec.json: { pretty: false }
```
DaemonSet mount secret chứng chỉ của KafkaUser `filebeat`, đặt resources limits, `securityContext` tối thiểu cần thiết (đọc `/var/log`).

---

## 5. Kiểm tra end-to-end
```bash
kubectl run logtest --image=busybox -n app --restart=Never -- sh -c "echo prod-log-test; sleep 2"
# Kafka nhận?
kubectl -n logging exec logging-kafka-broker-0 -- bin/kafka-console-consumer.sh \
  --topic k8s-logs --from-beginning --max-messages 1 \
  --bootstrap-server localhost:9093 --consumer.config /tmp/client.properties
# ES có index?  Kibana Discover (data view k8s-logs-*) thấy prod-log-test
```

---

## Tổng kết file 04

Luồng log production: Filebeat (TLS) → Kafka (Strimzi, mTLS+ACL, RF=3) → Logstash (PQ) → Elasticsearch (ECK, node roles, snapshot S3, ILM) → Kibana. Sang `05-dong-goi-app-pipeline.md`.
