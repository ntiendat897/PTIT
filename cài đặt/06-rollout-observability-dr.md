# 06 — Progressive delivery, observability & DR (production)

File này hoàn thiện các yêu cầu production còn lại: triển khai tăng tiến (canary) với **tự rollback theo SLO**, giám sát metrics (Prometheus/Grafana/Alertmanager), khôi phục thảm họa (DR), và cách đo **DORA** để đưa vào báo cáo.

---

## 1. Progressive delivery với Argo Rollouts (canary + auto-rollback)

Thay vì cập nhật toàn bộ cùng lúc, đưa phiên bản mới ra **từng phần** và tự động dừng/rollback nếu chỉ số xấu.

### 1.1 Cài Argo Rollouts
```bash
kubectl create namespace argo-rollouts
helm install argo-rollouts argo/argo-rollouts -n argo-rollouts
```

### 1.2 Thay Deployment bằng Rollout (canary) — trong Helm chart
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: myapp, namespace: app }
spec:
  replicas: 3
  selector: { matchLabels: { app: myapp } }
  template: { } # giống pod spec của Deployment (securityContext, probes, image@digest)
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 5m }
        - analysis:
            templates: [ { templateName: success-rate } ]
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

### 1.3 AnalysisTemplate dựa trên SLO (Prometheus) — tự rollback
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate, namespace: app }
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: "result >= 0.99"
      failureLimit: 2
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{app="myapp",code!~"5.."}[2m]))
            / sum(rate(http_requests_total{app="myapp"}[2m]))
```
Nếu tỉ lệ thành công < 99% → Argo Rollouts **tự dừng và rollback** về bản ổn định. Đây chính là cơ chế "tự bảo vệ production" để nêu trong báo cáo (MTTR rất thấp).

### 1.4 Rollback thủ công khi cần
```bash
kubectl argo rollouts get rollout myapp -n app        # xem tiến trình canary
kubectl argo rollouts undo myapp -n app               # rollback
```

> GitOps vẫn là nguồn sự thật: bản ổn định nằm trong Git; rollback nhanh nhất là `git revert` commit digest → Argo CD đồng bộ lại.

---

## 2. Observability — Prometheus + Grafana + Alertmanager

Production cần đủ **metrics + logs (đã có ELK ở file 04) + traces**.

### 2.1 Cài kube-prometheus-stack
```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.ingressClassName=nginx \
  --set 'grafana.ingress.hosts[0]=grafana.example.com' \
  --set grafana.ingress.tls[0].secretName=grafana-tls \
  --set 'grafana.ingress.tls[0].hosts[0]=grafana.example.com'
```
Bao gồm Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics. Mật khẩu Grafana lấy từ secret (đặt qua Vault); production nên bật SSO.

### 2.2 Cho ứng dụng xuất metrics
- App expose `/metrics` (định dạng Prometheus).
- Khai báo `ServiceMonitor` để Prometheus tự thu thập:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: myapp, namespace: app, labels: { release: kube-prometheus-stack } }
spec:
  selector: { matchLabels: { app: myapp } }
  endpoints: [ { port: http, path: /metrics, interval: 30s } ]
```

### 2.3 Cảnh báo (Alertmanager)
Định nghĩa `PrometheusRule` cho các SLO quan trọng (tỉ lệ lỗi 5xx, độ trễ p99, pod restart, CPU/mem) và route cảnh báo tới Slack/Email/PagerDuty. Đây là dữ liệu để tính **change failure rate** và **MTTR**.

### 2.4 (Tùy chọn nâng cao) Tracing
Cài OpenTelemetry Collector, app gắn SDK OTel, đẩy trace về Tempo/Jaeger — hoàn thiện bộ ba observability.

---

## 3. Khôi phục thảm họa (DR)

| Đối tượng | Cơ chế (đã thiết lập ở các file) | Kiểm thử khôi phục |
|---|---|---|
| etcd / cụm | etcd snapshot ra S3 (file 01) | Restore vào cụm thử định kỳ |
| Tài nguyên K8s | Velero backup hằng ngày (file 01) | `velero restore create --from-backup ...` |
| Elasticsearch | Snapshot ra S3 + ILM (file 04) | Restore snapshot vào cụm ES test |
| Harbor | PostgreSQL backup + image trên S3 (file 03) | Dựng lại Harbor trỏ DB/S3 đã backup |
| Git (nguồn sự thật GitOps) | Repo có bản sao/mirror | Clone lại, Argo CD dựng lại toàn bộ |

Nhờ GitOps, phần lớn cụm có thể **tái dựng từ Git + backup dữ liệu trạng thái**. Ghi lại RPO/RTO mục tiêu trong báo cáo.

---

## 4. Đo lường DORA (cho báo cáo)

| Chỉ số | Nguồn dữ liệu production |
|---|---|
| Lead time for changes | Khoảng cách commit (Git) → Argo CD `Synced/Healthy` (Argo CD API/lịch sử) |
| Deployment frequency | Số lần Argo CD sync thành công (Argo CD/Git history) |
| MTTR | Thời gian từ cảnh báo Alertmanager → rollback xong (Argo Rollouts/Prometheus) |
| Change failure rate | Số lần rollout bị analysis fail / tổng số rollout |

Lập **bảng so sánh trước (quy trình thủ công) / sau (CI/CD + GitOps)** cho bốn chỉ số, kèm bằng chứng (ảnh Argo CD, Grafana, Kibana, lịch sử Rollouts).

---

## 5. Checklist nghiệm thu production

- [ ] Cụm K8s HA (≥3 control-plane), etcd snapshot ra S3
- [ ] TLS hợp lệ cho Harbor/Jenkins/Kibana/Grafana/ArgoCD (cert-manager)
- [ ] Bí mật lấy từ Vault/ESO, không có mật khẩu phẳng trong Git
- [ ] Jenkins build rootless (Kaniko), ký Cosign, không mount docker.sock
- [ ] Harbor HA (DB/Redis/S3 ngoài), robot account, gating CVE, yêu cầu ký
- [ ] Kafka (Strimzi) TLS+SASL+ACL, RF=3, min.insync=2
- [ ] Elasticsearch node roles, snapshot S3, ILM bật
- [ ] Argo CD GitOps (auto-sync + selfHeal); Kyverno chặn image chưa ký
- [ ] Argo Rollouts canary + auto-rollback theo SLO
- [ ] Prometheus/Grafana/Alertmanager hoạt động, có cảnh báo SLO
- [ ] Velero backup + đã kiểm thử restore
- [ ] NetworkPolicy mặc định deny; Pod Security = restricted
- [ ] Đã thu thập số liệu DORA trước/sau

---

## Tổng kết

Hệ thống đạt chuẩn production: HA, bảo mật chuỗi cung ứng, GitOps, triển khai canary tự rollback, quan sát đầy đủ và DR. Đây là nền tảng vững để viết Chương 3–4 của báo cáo với số liệu định lượng thực tế.
