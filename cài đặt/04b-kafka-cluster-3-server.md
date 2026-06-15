# 04b — Cài đặt cụm Kafka trên 3 server Ubuntu 24.04 (production)

Hướng dẫn dựng **cụm Apache Kafka 4.2.0** (KRaft, không ZooKeeper) trên **3 server vật lý/VM Ubuntu 24.04**, chuẩn production: mã hóa **TLS**, xác thực **SASL/SCRAM-SHA-512**, phân quyền **ACL**, chạy bằng **systemd**, có **giám sát Prometheus** và **giao diện quản trị**.

> Đây là phương án Kafka **chạy trực tiếp trên máy chủ** (thay cho phương án Strimzi-trên-Kubernetes ở file `04`). Nếu dùng file này, phần ELK/Logstash/Filebeat ở file `04` sẽ trỏ tới cụm Kafka ngoài này (xem mục cuối).

## Phiên bản (ổn định mới nhất, 6/2026)

| Phần mềm | Phiên bản | Ghi chú |
|---|---|---|
| Apache Kafka | 4.2.0 (Scala 2.13) | KRaft-only, broker cần Java 17+ |
| Java (Temurin JDK) | 21 LTS | LTS, khuyến nghị cho broker |
| JMX Prometheus exporter | mới nhất | giám sát |
| AKHQ (UI quản trị) | mới nhất | tùy chọn |

## Mô hình triển khai

3 node **combined** (mỗi node vừa là `controller` vừa là `broker`) → quorum controller 3, chịu lỗi 1 node.

| Node | Hostname | IP (ví dụ) | node.id |
|---|---|---|---|
| 1 | kafka-1 | 10.0.0.11 | 1 |
| 2 | kafka-2 | 10.0.0.12 | 2 |
| 3 | kafka-3 | 10.0.0.13 | 3 |

Cổng dùng:

| Cổng | Listener | Giao thức | Vai trò |
|---|---|---|---|
| 9092 | CLIENT | SASL_SSL | client (Filebeat, Logstash, app) kết nối |
| 9094 | INTERNAL | SASL_SSL | broker ↔ broker |
| 9093 | CONTROLLER | SSL (mTLS) | quorum controller |
| 9404 | — | HTTP | JMX exporter cho Prometheus |

> Quy mô lớn hơn nên tách controller và broker thành các node riêng (isolated mode). Với đúng 3 server, combined mode là lựa chọn hợp lý và được hỗ trợ chính thức.

---

## 1. Chuẩn bị 3 server (làm trên CẢ 3 node)

### 1.1 Hostname & phân giải tên
Đặt hostname từng máy:
```bash
sudo hostnamectl set-hostname kafka-1     # kafka-2, kafka-3 tương ứng
```
Thêm vào `/etc/hosts` trên cả 3 máy:
```bash
sudo tee -a /etc/hosts > /dev/null <<EOF
10.0.0.11  kafka-1
10.0.0.12  kafka-2
10.0.0.13  kafka-3
EOF
```

### 1.2 Đồng bộ thời gian (bắt buộc cho cụm phân tán)
```bash
sudo apt-get update && sudo apt-get install -y chrony
sudo systemctl enable --now chrony
timedatectl   # kiểm tra "System clock synchronized: yes"
```

### 1.3 Người dùng & thư mục riêng cho Kafka
```bash
sudo useradd -r -m -d /var/lib/kafka -s /usr/sbin/nologin kafka
sudo mkdir -p /opt/kafka /var/lib/kafka/data /var/log/kafka /etc/kafka/ssl
sudo chown -R kafka:kafka /var/lib/kafka /var/log/kafka /etc/kafka
```
> Production: gắn `/var/lib/kafka/data` lên ổ đĩa/volume riêng (SSD), filesystem `xfs` hoặc `ext4`, mount với `noatime`.

### 1.4 Giới hạn hệ thống (file descriptors, mmap)
```bash
sudo tee /etc/security/limits.d/kafka.conf > /dev/null <<EOF
kafka  soft  nofile  100000
kafka  hard  nofile  100000
kafka  soft  nproc   32768
kafka  hard  nproc   32768
EOF

sudo tee /etc/sysctl.d/99-kafka.conf > /dev/null <<EOF
vm.swappiness=1
vm.max_map_count=262144
net.core.somaxconn=1024
EOF
sudo sysctl --system
```

### 1.5 Tường lửa (mở cổng nội cụm)
```bash
sudo apt-get install -y ufw
# Cho phép 3 node nói chuyện với nhau + cổng client
for ip in 10.0.0.11 10.0.0.12 10.0.0.13; do
  sudo ufw allow from $ip to any port 9092,9093,9094 proto tcp
done
# Client (app/ELK) + Prometheus — giới hạn theo subnet thực tế của bạn
sudo ufw allow from 10.0.0.0/24 to any port 9092 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 9404 proto tcp
sudo ufw enable
```

---

## 2. Cài Java 21 (CẢ 3 node)

```bash
sudo apt-get install -y wget gnupg
# Eclipse Temurin 21 (Adoptium)
sudo mkdir -p /etc/apt/keyrings
wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public | \
  sudo gpg --dearmor -o /etc/apt/keyrings/adoptium.gpg
echo "deb [signed-by=/etc/apt/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb $(. /etc/os-release; echo $VERSION_CODENAME) main" | \
  sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt-get update
sudo apt-get install -y temurin-21-jdk
java -version    # phải hiển thị 21.x
```

---

## 3. Cài Kafka 4.2.0 (CẢ 3 node)

```bash
cd /tmp
KAFKA_VER=4.2.0
SCALA_VER=2.13
wget https://downloads.apache.org/kafka/${KAFKA_VER}/kafka_${SCALA_VER}-${KAFKA_VER}.tgz
# (Khuyến nghị) kiểm tra chữ ký/sha512 từ trang Apache trước khi cài
tar xzf kafka_${SCALA_VER}-${KAFKA_VER}.tgz
sudo cp -r kafka_${SCALA_VER}-${KAFKA_VER}/* /opt/kafka/
sudo chown -R kafka:kafka /opt/kafka
/opt/kafka/bin/kafka-topics.sh --version    # 4.2.0
```

---

## 4. Sinh chứng chỉ TLS (làm 1 lần trên máy quản trị, rồi phân phối)

Tạo CA nội bộ và keystore/truststore (PKCS12) cho từng node, có **SAN** đúng hostname + IP.

```bash
mkdir -p ~/kafka-ca && cd ~/kafka-ca
# Mật khẩu keystore/truststore (đặt mạnh, giữ bí mật)
STOREPASS='ChangeMe_Store_Pass'

# 4.1 Tạo CA
openssl req -new -x509 -days 3650 -nodes \
  -keyout ca.key -out ca.crt -subj "/CN=Kafka-Internal-CA"

# 4.2 Truststore chung (chứa CA) cho mọi node & client
keytool -keystore kafka.truststore.p12 -storetype PKCS12 -alias CA \
  -import -file ca.crt -storepass "$STOREPASS" -noprompt

# 4.3 Keystore cho từng broker (lặp cho kafka-1/2/3)
for i in 1 2 3; do
  HOST="kafka-$i"; IP="10.0.0.1$i"
  keytool -keystore ${HOST}.keystore.p12 -storetype PKCS12 -alias ${HOST} \
    -validity 3650 -genkey -keyalg RSA -storepass "$STOREPASS" \
    -dname "CN=${HOST}" -ext "SAN=DNS:${HOST},IP:${IP}"
  # Tạo CSR, ký bằng CA, nạp lại
  keytool -keystore ${HOST}.keystore.p12 -alias ${HOST} -certreq -file ${HOST}.csr \
    -storepass "$STOREPASS"
  openssl x509 -req -CA ca.crt -CAkey ca.key -in ${HOST}.csr -out ${HOST}.signed.crt \
    -days 3650 -CAcreateserial -extfile <(printf "subjectAltName=DNS:${HOST},IP:${IP}")
  keytool -keystore ${HOST}.keystore.p12 -alias CA -import -file ca.crt \
    -storepass "$STOREPASS" -noprompt
  keytool -keystore ${HOST}.keystore.p12 -alias ${HOST} -import -file ${HOST}.signed.crt \
    -storepass "$STOREPASS" -noprompt
done
```

Phân phối tới mỗi node (đặt trong `/etc/kafka/ssl`):
```bash
# Ví dụ cho kafka-1
scp kafka-1.keystore.p12 kafka.truststore.p12 user@10.0.0.11:/tmp/
# Trên kafka-1:
sudo mv /tmp/kafka-1.keystore.p12 /etc/kafka/ssl/kafka.keystore.p12
sudo mv /tmp/kafka.truststore.p12 /etc/kafka/ssl/kafka.truststore.p12
sudo chown kafka:kafka /etc/kafka/ssl/*.p12
sudo chmod 600 /etc/kafka/ssl/*.p12
```

> Production: nên dùng chứng chỉ từ CA nội bộ của công ty thay vì tự ký, và lưu mật khẩu keystore qua config provider/Vault (KIP-297) thay vì để phẳng trong file.

---

## 5. Cấu hình `server.properties` (mỗi node một bản)

Tạo `/etc/kafka/server.properties`. Phần lớn giống nhau, chỉ khác `node.id` và `advertised.listeners`/`SAN`. Dưới đây là bản cho **kafka-1** (đổi `node.id`, các `kafka-1` thành hostname tương ứng cho node 2, 3):

```properties
############ KRaft ############
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093

############ Listeners ############
listeners=CLIENT://:9092,INTERNAL://:9094,CONTROLLER://:9093
advertised.listeners=CLIENT://kafka-1:9092,INTERNAL://kafka-1:9094
inter.broker.listener.name=INTERNAL
controller.listener.names=CONTROLLER
listener.security.protocol.map=CLIENT:SASL_SSL,INTERNAL:SASL_SSL,CONTROLLER:SSL

############ TLS ############
ssl.keystore.location=/etc/kafka/ssl/kafka.keystore.p12
ssl.keystore.password=ChangeMe_Store_Pass
ssl.keystore.type=PKCS12
ssl.truststore.location=/etc/kafka/ssl/kafka.truststore.p12
ssl.truststore.password=ChangeMe_Store_Pass
ssl.truststore.type=PKCS12
ssl.endpoint.identification.algorithm=HTTPS
# Controller (mTLS): bắt buộc client cert giữa các controller
ssl.client.auth=required

############ SASL/SCRAM ############
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
listener.name.internal.sasl.enabled.mechanisms=SCRAM-SHA-512
listener.name.client.sasl.enabled.mechanisms=SCRAM-SHA-512
# JAAS nội tuyến cho inter-broker (broker đăng nhập bằng user admin)
listener.name.internal.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="ChangeMe_Admin_Pass";

############ Authorization (ACL) ############
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:admin
allow.everyone.if.no.acl.found=false

############ Độ bền & sao chép (production) ############
default.replication.factor=3
min.insync.replicas=2
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
num.partitions=6
auto.create.topics.enable=false
unclean.leader.election.enable=false

############ Lưu trữ & log ############
log.dirs=/var/lib/kafka/data
metadata.log.dir=/var/lib/kafka/data
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

############ Hiệu năng ############
num.network.threads=6
num.io.threads=8
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
socket.request.max.bytes=104857600
```

> Đặt quyền chặt cho file vì có mật khẩu: `sudo chown kafka:kafka /etc/kafka/server.properties && sudo chmod 600 /etc/kafka/server.properties`.

---

## 6. Heap & biến môi trường

```bash
sudo tee /etc/kafka/kafka.env > /dev/null <<'EOF'
KAFKA_HEAP_OPTS=-Xmx6g -Xms6g
KAFKA_JVM_PERFORMANCE_OPTS=-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35
LOG_DIR=/var/log/kafka
EOF
```
> Quy tắc: heap 6 GB cho máy 16–32 GB RAM; phần RAM còn lại để OS page cache (rất quan trọng với Kafka). Không đặt heap quá lớn.

---

## 7. Format ổ lưu trữ KRaft (khởi tạo cụm)

Tạo **một** cluster UUID dùng chung, và bootstrap sẵn user `admin` (SCRAM) để inter-broker đăng nhập được ngay.

```bash
# (Chạy 1 lần trên kafka-1) sinh UUID
sudo -u kafka /opt/kafka/bin/kafka-storage.sh random-uuid
# Ví dụ kết quả: Az>5kq9ZQ-OBC0d... → đặt cho biến dưới, DÙNG CHUNG cho cả 3 node
export KAFKA_CLUSTER_ID="<UUID_vua_sinh>"
```

Trên **mỗi node** (dùng cùng `KAFKA_CLUSTER_ID`):
```bash
sudo -u kafka KAFKA_CLUSTER_ID="$KAFKA_CLUSTER_ID" /opt/kafka/bin/kafka-storage.sh format \
  -t "$KAFKA_CLUSTER_ID" \
  -c /etc/kafka/server.properties \
  --add-scram 'SCRAM-SHA-512=[name=admin,password=ChangeMe_Admin_Pass]'
```
`--add-scram` ghi sẵn thông tin user `admin` vào metadata → broker xác thực SASL nội bộ ngay từ lần khởi động đầu.

---

## 8. Tạo dịch vụ systemd (mỗi node)

```bash
sudo tee /etc/systemd/system/kafka.service > /dev/null <<'EOF'
[Unit]
Description=Apache Kafka (KRaft)
Requires=network.target
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka
EnvironmentFile=/etc/kafka/kafka.env
ExecStart=/opt/kafka/bin/kafka-server-start.sh /etc/kafka/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=5
LimitNOFILE=100000
TimeoutStopSec=180

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now kafka
```

**Kiểm tra (mỗi node):**
```bash
sudo systemctl status kafka --no-pager
sudo journalctl -u kafka -f          # theo dõi log khởi động
```

---

## 9. Kiểm tra cụm & quorum

Tạo file cấu hình client admin để gọi lệnh (dùng SASL_SSL):
```bash
sudo tee /etc/kafka/admin.properties > /dev/null <<EOF
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="ChangeMe_Admin_Pass";
ssl.truststore.location=/etc/kafka/ssl/kafka.truststore.p12
ssl.truststore.password=ChangeMe_Store_Pass
ssl.truststore.type=PKCS12
EOF
sudo chmod 600 /etc/kafka/admin.properties
```

```bash
# Trạng thái quorum controller (phải thấy 3 voter, có 1 leader)
/opt/kafka/bin/kafka-metadata-quorum.sh \
  --bootstrap-server kafka-1:9092 --command-config /etc/kafka/admin.properties describe --status

# Danh sách broker
/opt/kafka/bin/kafka-broker-api-versions.sh \
  --bootstrap-server kafka-1:9092 --command-config /etc/kafka/admin.properties | grep id
```

---

## 10. Tạo user SCRAM & ACL cho client (Filebeat, Logstash, app)

Tạo user riêng cho mỗi client (không dùng admin):
```bash
# User cho Filebeat (chỉ ghi) và Logstash (chỉ đọc)
/opt/kafka/bin/kafka-configs.sh --bootstrap-server kafka-1:9092 --command-config /etc/kafka/admin.properties \
  --alter --add-config 'SCRAM-SHA-512=[password=FilebeatPass]' --entity-type users --entity-name filebeat
/opt/kafka/bin/kafka-configs.sh --bootstrap-server kafka-1:9092 --command-config /etc/kafka/admin.properties \
  --alter --add-config 'SCRAM-SHA-512=[password=LogstashPass]' --entity-type users --entity-name logstash
```

Tạo topic log và cấp ACL tối thiểu:
```bash
/opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka-1:9092 --command-config /etc/kafka/admin.properties \
  --create --topic k8s-logs --partitions 6 --replication-factor 3

# filebeat: chỉ Write/Describe topic
/opt/kafka/bin/kafka-acls.sh --bootstrap-server kafka-1:9092 --command-config /etc/kafka/admin.properties \
  --add --allow-principal User:filebeat --operation Write --operation Describe --topic k8s-logs
# logstash: Read topic + dùng consumer group
/opt/kafka/bin/kafka-acls.sh --bootstrap-server kafka-1:9092 --command-config /etc/kafka/admin.properties \
  --add --allow-principal User:logstash --operation Read --operation Describe --topic k8s-logs
/opt/kafka/bin/kafka-acls.sh --bootstrap-server kafka-1:9092 --command-config /etc/kafka/admin.properties \
  --add --allow-principal User:logstash --operation Read --group logstash
```

**Kiểm tra gửi/nhận:**
```bash
# Producer thử (Ctrl-D để kết thúc)
/opt/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka-1:9092 \
  --producer.config /etc/kafka/admin.properties --topic k8s-logs
# Consumer thử
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-1:9092 \
  --consumer.config /etc/kafka/admin.properties --topic k8s-logs --from-beginning --max-messages 1
```

---

## 11. Giám sát: JMX Prometheus exporter (mỗi node)

```bash
sudo mkdir -p /opt/kafka/jmx
cd /opt/kafka/jmx
# Lấy bản jmx_exporter javaagent mới nhất từ GitHub
JMX_VER=$(curl -s https://api.github.com/repos/prometheus/jmx_exporter/releases/latest | grep -oP '"tag_name": "\K[^"]+' | tr -d v)
sudo wget -O jmx_prometheus_javaagent.jar \
  https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/${JMX_VER}/jmx_prometheus_javaagent-${JMX_VER}.jar
# Lấy cấu hình mẫu cho Kafka
sudo wget -O kafka.yml https://raw.githubusercontent.com/prometheus/jmx_exporter/main/example_configs/kafka-kraft.yml
```

Gắn javaagent vào Kafka — thêm vào `/etc/kafka/kafka.env`:
```bash
echo 'KAFKA_OPTS=-javaagent:/opt/kafka/jmx/jmx_prometheus_javaagent.jar=9404:/opt/kafka/jmx/kafka.yml' | sudo tee -a /etc/kafka/kafka.env
sudo systemctl restart kafka
curl -s http://localhost:9404/metrics | head    # thấy metric kafka_*
```
Khai báo 3 target `kafka-{1,2,3}:9404` trong Prometheus (file 06) và nhập dashboard Kafka có sẵn vào Grafana.

---

## 12. Phần mềm bổ trợ (tùy chọn)

### AKHQ — giao diện quản trị Kafka
Chạy bằng Docker trên một máy quản trị (kết nối SASL_SSL tới cụm):
```bash
docker run -d --name akhq -p 8080:8080 \
  -v /etc/kafka/ssl/kafka.truststore.p12:/app/truststore.p12:ro \
  -e AKHQ_CONFIGURATION="$(cat <<'YAML'
akhq:
  connections:
    prod-kafka:
      properties:
        bootstrap.servers: "kafka-1:9092,kafka-2:9092,kafka-3:9092"
        security.protocol: SASL_SSL
        sasl.mechanism: SCRAM-SHA-512
        sasl.jaas.config: org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="ChangeMe_Admin_Pass";
        ssl.truststore.location: /app/truststore.p12
        ssl.truststore.password: ChangeMe_Store_Pass
YAML
)" tailscale/akhq:latest 2>/dev/null || echo "Dùng image tchiotludo/akhq:latest"
```
> Image chính thức: `tchiotludo/akhq`. Đặt sau reverse proxy có TLS + xác thực khi mở cho người dùng.

### kcat — công cụ dòng lệnh test nhanh
```bash
sudo apt-get install -y kcat
kcat -b kafka-1:9092 -L \
  -X security.protocol=SASL_SSL -X sasl.mechanism=SCRAM-SHA-512 \
  -X sasl.username=admin -X sasl.password=ChangeMe_Admin_Pass \
  -X ssl.ca.location=/etc/kafka/ssl/ca.crt
```

---

## 13. Vận hành production

- **Rolling restart**: restart từng node một, đợi `under-replicated-partitions = 0` trước khi sang node kế:
  ```bash
  /opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka-1:9092 \
    --command-config /etc/kafka/admin.properties --describe --under-replicated-partitions
  ```
- **Theo dõi sức khỏe**: `under-replicated-partitions = 0`, `offline-partitions = 0`, quorum có leader.
- **Sao lưu metadata**: backup định kỳ thư mục `/var/lib/kafka/data` (đặc biệt metadata KRaft); kiểm thử khôi phục.
- **Gia hạn chứng chỉ** trước khi hết hạn; theo dõi dung lượng đĩa (`log.retention`).
- **Cảnh báo** (Alertmanager): URP > 0, broker offline, disk > 80%, ISR shrink.

**Kiểm tra cuối (checklist):**
- [ ] 3 broker online, quorum controller có 1 leader + 3 voter
- [ ] Topic test RF=3, `min.insync.replicas=2`, URP=0
- [ ] Client kết nối qua SASL_SSL (không có cổng PLAINTEXT mở)
- [ ] ACL chặn user không phận sự (`allow.everyone.if.no.acl.found=false`)
- [ ] Metric `kafka_*` xuất ra Prometheus ở cả 3 node
- [ ] Tường lửa chỉ mở cổng cần thiết theo subnet

---

## 14. Kết nối với hệ thống log (file 04)

Nếu dùng cụm Kafka này thay cho Strimzi, hãy trỏ Filebeat và Logstash tới đây (đổi cấu hình ở file `04`):

- **Filebeat** `output.kafka.hosts: ["kafka-1:9092","kafka-2:9092","kafka-3:9092"]`, dùng `security.protocol: SASL_SSL`, `sasl.mechanism: SCRAM-SHA-512`, user `filebeat`, kèm `ssl.certificate_authorities: ["ca.crt"]`.
- **Logstash** input kafka: `bootstrap_servers` như trên, `security_protocol => "SASL_SSL"`, `sasl_mechanism => "SCRAM-SHA-512"`, JAAS user `logstash`, truststore CA.

---

## Tổng kết

Bạn đã có cụm Kafka 4.2.0 KRaft 3 node chuẩn production: TLS + SASL/SCRAM + ACL, RF=3/min.insync=2, chạy bằng systemd, giám sát qua Prometheus và quản trị qua AKHQ. Đây là nền tảng hàng đợi log (và message bus) đáng tin cậy cho hệ thống.
