# 04 — Giám sát log: Elasticsearch cluster + Kibana + Logstash (Ubuntu 24.04)

Luồng log: **Filebeat (trên K8s) → Kafka (cụm 3 server, file `04b`) → Logstash → Elasticsearch (cụm) → Kibana**. File này cài **Elasticsearch cluster**, **Kibana** và **Logstash** trên các server riêng (Ubuntu 24.04), bật **bảo mật (TLS + xác thực)**.

> Các server đã làm cứng theo file 01. Mở cổng nội cụm giới hạn theo subnet.

```bash
ES_VER=9.4.2        # đồng bộ phiên bản trên mọi node Elastic
```

---

## 1. Cài Elasticsearch (cụm 3 node)

Trên **mỗi node ES** (es-1/2/3):
```bash
sudo apt install -y gnupg apt-transport-https
wget -qO- https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
sudo apt update && sudo apt install -y elasticsearch=$ES_VER
```

Tối ưu heap (50% RAM, tối đa 31GB) — `/etc/elasticsearch/jvm.options.d/heap.options`:
```
-Xms8g
-Xmx8g
```
Cấu hình `/etc/elasticsearch/elasticsearch.yml` (đổi tên/IP từng node):
```yaml
cluster.name: vnpt-logs
node.name: es-1
node.roles: [ master, data, ingest ]
path.data: /data/elasticsearch          # ổ riêng, noatime
path.logs: /var/log/elasticsearch
network.host: 10.0.1.11
discovery.seed_hosts: ["10.0.1.11","10.0.1.12","10.0.1.13"]
cluster.initial_master_nodes: ["es-1","es-2","es-3"]
# Bảo mật (bật mặc định 8.x/9.x): TLS transport + http
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```
Sinh chứng chỉ CA + node (chạy một lần, phân phối):
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca --out /etc/elasticsearch/certs/ca.p12
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca /etc/elasticsearch/certs/ca.p12 --out /etc/elasticsearch/certs/node.p12
# Khai báo keystore/truststore p12 cho transport + http trong elasticsearch.yml
```
```bash
sudo systemctl enable --now elasticsearch
# Đặt mật khẩu user dựng sẵn
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```
**Kiểm tra:** `curl -k -u elastic https://10.0.1.11:9200/_cluster/health` → status green, 3 node.

> Tối ưu: tách ổ data SSD; bật **ILM** để xoay vòng/giữ log (vd hot 7 ngày → xóa 30 ngày), tránh đầy đĩa.

---

## 2. Cài Kibana (server riêng)

```bash
sudo apt install -y kibana=$ES_VER
```
`/etc/kibana/kibana.yml`:
```yaml
server.host: "0.0.0.0"
server.publicBaseUrl: "https://kibana.vnpt.vn"
elasticsearch.hosts: ["https://10.0.1.11:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "<mật khẩu>"
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/ca.crt"]
# TLS cho Kibana
server.ssl.enabled: true
server.ssl.certificate: /etc/ssl/vnpt/kibana.crt
server.ssl.key: /etc/ssl/vnpt/kibana.key
```
```bash
sudo systemctl enable --now kibana
sudo ufw allow from 10.0.0.0/24 to any port 5601 proto tcp
```
Truy cập `https://kibana.vnpt.vn:5601`. Production nên cấu hình **SSO** cho Kibana.

---

## 3. Cài Logstash (đọc Kafka → ghi Elasticsearch)

Trên server Logstash:
```bash
sudo apt install -y logstash=$ES_VER
```
`/etc/logstash/conf.d/k8s-logs.conf`:
```
input {
  kafka {
    bootstrap_servers => "kafka-1:9093,kafka-2:9093,kafka-3:9093"
    topics => ["k8s-logs"]
    group_id => "logstash"
    codec => json
    security_protocol => "SSL"
    ssl_truststore_location => "/etc/logstash/certs/kafka-ca.p12"
    ssl_keystore_location => "/etc/logstash/certs/logstash-user.p12"
  }
}
filter {
  if [kubernetes] { mutate { add_field => { "pod" => "%{[kubernetes][pod][name]}" "ns" => "%{[kubernetes][namespace]}" } } }
}
output {
  elasticsearch {
    hosts => ["https://10.0.1.11:9200"]
    user => "logstash_writer"
    password => "<mật khẩu>"
    ssl_certificate_authorities => "/etc/logstash/certs/es-ca.crt"
    ilm_enabled => true
    ilm_rollover_alias => "k8s-logs"
    ilm_policy => "k8s-logs-policy"
  }
}
```
Bật **persistent queue** trong `/etc/logstash/logstash.yml` (`queue.type: persisted`) để không mất sự kiện. Heap Logstash ~1–2GB.
```bash
sudo systemctl enable --now logstash
```

---

## 4. Filebeat trên Kubernetes → Kafka

Filebeat chạy **DaemonSet trên cụm K8s**, thu log pod và đẩy vào Kafka (cụm `04b`) qua TLS + chứng chỉ KafkaUser `filebeat`. Cấu hình output Kafka:
```yaml
output.kafka:
  hosts: ["kafka-1:9093","kafka-2:9093","kafka-3:9093"]
  topic: "k8s-logs"
  ssl.certificate_authorities: ["/certs/ca.crt"]
  ssl.certificate: "/certs/user.crt"
  ssl.key: "/certs/user.key"
  codec.json: {}
```
(Manifest DaemonSet đầy đủ + tạo KafkaUser xem file `04b` mục 10 và 14.)

---

## 5. Kiểm tra end-to-end
```bash
kubectl run logtest -n app --image=busybox --restart=Never -- sh -c "echo vnpt-log-test; sleep 2"
```
Kafka nhận → Logstash xử lý → ES có index `k8s-logs-*` → tìm `vnpt-log-test` trên **Kibana Discover**.

---

## Tổng kết file 04
Cụm ELK (Elasticsearch + Kibana + Logstash) chạy nội bộ, bật bảo mật TLS + xác thực, đọc log từ cụm Kafka. Cụm Kafka xem `04b-kafka-cluster-3-server.md`. Sang `05-dong-goi-app-pipeline.md`.
