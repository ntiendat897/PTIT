# 06 — Giám sát metrics (Prometheus + Grafana), cảnh báo & DR

File này hoàn thiện phần quan sát metrics, cảnh báo (kết hợp Telegram), sao lưu/khôi phục thảm họa (DR) và checklist nghiệm thu production. Log đã có ở file 04 (Kafka–ELK).

---

## 1. Prometheus + Grafana trên cụm K8s (kube-prometheus-stack)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.ingress.enabled=true --set grafana.ingress.ingressClassName=nginx \
  --set 'grafana.ingress.hosts[0]=grafana.vnpt.vn' \
  --set grafana.ingress.tls[0].secretName=grafana-tls \
  --set 'grafana.ingress.tls[0].hosts[0]=grafana.vnpt.vn' \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```
Bao gồm Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics. Mật khẩu Grafana đặt qua secret; production nên bật **SSO/LDAP**.

> Giám sát các server VM ngoài K8s (Jenkins, Harbor, Kafka, ES): cài **node_exporter** trên từng VM và khai báo target tĩnh cho Prometheus; Kafka/ES dùng JMX/Elasticsearch exporter (xem file 04b mục 11 cho Kafka JMX).

---

## 2. Ứng dụng xuất metrics

App expose `/metrics` (Prometheus format) và khai báo `ServiceMonitor`:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: api-myvnpt-config, namespace: app, labels: { release: kube-prometheus-stack } }
spec:
  selector: { matchLabels: { app: api-myvnpt-config } }
  endpoints: [ { port: http, path: /metrics, interval: 30s } ]
```

---

## 3. Cảnh báo (Alertmanager → Telegram)

Định nghĩa `PrometheusRule` cho các SLO quan trọng (tỉ lệ lỗi 5xx, độ trễ p99, pod restart, CPU/mem/đĩa cao, target down). Cấu hình **Alertmanager gửi cảnh báo qua Telegram** để đồng nhất với kênh thông báo build:
```yaml
receivers:
  - name: telegram
    telegram_configs:
      - bot_token: "<telegram-bot-token>"
        chat_id: <CHAT_ID>
        api_url: "https://api.telegram.org"
route: { receiver: telegram, group_wait: 30s, repeat_interval: 4h }
```
> Token đặt qua secret của Alertmanager, không ghi phẳng.

---

## 4. Sao lưu & khôi phục thảm họa (DR)

| Đối tượng | Cơ chế | Kiểm thử khôi phục |
|---|---|---|
| etcd / cụm K8s | Snapshot etcd ra lưu trữ nội bộ (file 01) | Restore vào cụm thử |
| Tài nguyên K8s | **Velero** backup hằng ngày | `velero restore create --from-backup ...` |
| Elasticsearch | Snapshot repository + ILM (file 04) | Restore snapshot vào ES test |
| Harbor | Backup `/data` (DB + image) (file 03) | Dựng lại Harbor từ backup |
| Kafka | Backup metadata + cấu hình (file 04b) | Khôi phục theo quy trình rolling |
| Git (devsecops, app) | Mirror/clone định kỳ | Clone lại; Argo CD dựng lại cụm |

Cài Velero (backup tài nguyên K8s ra object storage nội bộ S3-compatible):
```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts && helm repo update
helm install velero vmware-tanzu/velero -n velero --create-namespace \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=velero-backup \
  --set configuration.backupStorageLocation[0].config.s3Url=https://s3.noi-bo.vnpt.vn
velero schedule create daily --schedule "0 2 * * *" --ttl 720h0m0s
```
Ghi rõ **RPO/RTO** mục tiêu và **kiểm thử khôi phục định kỳ**.

---

## 5. Checklist nghiệm thu production

- [ ] Mọi server làm cứng (SSH key-only, ufw deny mặc định, fail2ban, auto-update); chỉ JUMP truy cập quản trị
- [ ] TLS nội bộ (CA VNPT) cho Harbor/Jenkins/Kibana/Grafana/Argo CD/Kafka/Elasticsearch
- [ ] Bí mật lấy từ credential Jenkins/secret, không ghi phẳng
- [ ] Cụm K8s HA (≥3 control-plane), etcd snapshot
- [ ] Harbor tự quét Trivy, chặn image vượt ngưỡng High; robot account least-privilege
- [ ] Kafka TLS+SASL+ACL (RF=3, min.insync=2); ELK bật bảo mật + ILM
- [ ] Pipeline kích hoạt bằng tag → build Maven → Sonar → push Harbor → cập nhật Helm Chart devsecops
- [ ] Argo CD auto-sync + selfHeal; rollback qua History and Rollback
- [ ] Prometheus/Grafana hoạt động; cảnh báo SLO gửi Telegram; thông báo build Telegram
- [ ] Velero backup + đã kiểm thử restore; etcd/ES/Harbor backup
- [ ] NetworkPolicy mặc định deny; Pod Security = restricted
- [ ] Đã thu thập số liệu DORA trước/sau

---

## 6. (Nâng cao — hướng phát triển)

Các hạng mục tăng cường ngoài luồng hiện tại: triển khai tăng tiến **canary/blue-green tự rollback theo SLO** bằng Argo Rollouts; **ký – xác minh image** (Cosign) + **kiểm soát chấp nhận** (Kyverno) chỉ chạy image đã ký; quản lý bí mật tập trung (Vault); đăng nhập một lần (Keycloak/OIDC); AIOps phân tích log.

---

## Tổng kết
Hệ thống đạt chuẩn production nội bộ: làm cứng toàn diện trên Ubuntu 24.04, CI/CD theo đúng luồng VNPT (tag → Maven → Harbor/Trivy → devsecops → Argo CD), giám sát log + metrics, cảnh báo và DR đầy đủ.
