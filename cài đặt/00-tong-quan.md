# 00 — Tổng quan triển khai (chuẩn production)

Bộ tài liệu này hướng dẫn triển khai **chuẩn production** cho đề tài *"Tự động hóa triển khai phần mềm kết hợp giám sát log tập trung"*, dùng làm cơ sở viết báo cáo thực tập tốt nghiệp (thạc sĩ).

> Đây là môi trường **production thật**, không phải lab. Mọi thành phần đều bật bảo mật (TLS, xác thực, phân quyền), có tính sẵn sàng cao (HA), sao lưu/khôi phục (DR) và giám sát đầy đủ. Các "lối tắt lab" (HTTP, tắt security, single-node, mount docker.sock…) đều bị loại bỏ.

## Kiến trúc tổng thể

```
Dev → Git (app repo) → Jenkins CI:
        build rootless (Kaniko) → test → quét bảo mật (Trivy) → ký image (Cosign)
        → push Harbor → cập nhật image tag ở Git (manifests repo)

Argo CD (GitOps) theo dõi manifests repo → đồng bộ vào K8s → Argo Rollouts (canary/blue-green)
        → Kyverno chỉ cho chạy image đã ký & đã quét

Quan sát:  Pod → Filebeat → Kafka (Strimzi, TLS/SASL) → Logstash → Elasticsearch (ECK) → Kibana
           Pod/cluster → Prometheus → Grafana + Alertmanager
```

Hai nguyên tắc cốt lõi:
1. **GitOps**: Git là nguồn sự thật duy nhất; mọi thay đổi trạng thái cụm đều qua commit, Argo CD đồng bộ. Jenkins **không** `kubectl apply`/`helm upgrade` trực tiếp lên production.
2. **Supply-chain security**: image được quét + ký; cụm chỉ chạy image đã ký qua chính sách Kyverno.

## Nguyên tắc production áp dụng xuyên suốt

- **TLS ở mọi nơi**: Harbor, Jenkins, Kibana, Grafana qua Ingress + cert-manager; Kafka/Elasticsearch mã hóa nội bộ.
- **Bí mật tập trung**: dùng External Secrets Operator + Vault (hoặc Sealed Secrets), tuyệt đối không để mật khẩu phẳng trong Git/manifest.
- **Least privilege**: RBAC chặt, ServiceAccount riêng từng workload, robot account cho CI.
- **HA & dung lượng**: thành phần trạng thái (etcd, Harbor DB, Kafka, Elasticsearch) chạy nhiều bản sao, có StorageClass bền.
- **Sao lưu & DR**: Velero cho cụm, etcd snapshot, snapshot Elasticsearch và Harbor ra object storage (S3).
- **Quan sát đủ bộ ba**: metrics (Prometheus), logs (ELK), và sẵn sàng cho traces (OpenTelemetry).
- **Resource governance**: requests/limits, HPA, PodDisruptionBudget, NetworkPolicy cho mọi workload.

## Yêu cầu môi trường

| Hạng mục | Yêu cầu production |
|---|---|
| Kubernetes | Cụm conformant, nhiều node (≥3 control-plane HA + worker), phiên bản ≥ 1.31 |
| StorageClass | Hỗ trợ `ReadWriteOnce` bền (CSI), có snapshot |
| Object storage | S3-compatible (cho backup ES/Harbor/Velero) |
| DNS | Tên miền nội bộ/khả tín cho Harbor, Jenkins, Kibana, Grafana, ArgoCD |
| CA/Chứng chỉ | Let's Encrypt hoặc CA nội bộ qua cert-manager |
| Quản lý bí mật | Vault hoặc giải pháp tương đương |

## Thứ tự thực hiện

| File | Nội dung |
|---|---|
| `01-ha-tang-nen.md` | Cụm K8s HA, platform services: ingress-nginx, cert-manager, External Secrets/Vault, StorageClass, Velero, namespaces & RBAC |
| `02-jenkins.md` | Jenkins CI trên K8s (agent động, build rootless Kaniko, ký Cosign), HTTPS, secrets từ Vault |
| `03-registry-va-trivy.md` | Harbor HA (Postgres/Redis/S3 ngoài, TLS, OIDC, robot account, gating, ký) + Trivy |
| `04-kafka-elk-cluster.md` | Kafka qua Strimzi (TLS/SASL/ACL), ELK qua ECK (node roles, snapshot S3, ILM), Logstash, Filebeat |
| `05-dong-goi-app-pipeline.md` | Dockerfile hardened, Helm chart production, Jenkinsfile CI, ArgoCD Application, Kyverno |
| `06-rollout-observability-dr.md` | Argo Rollouts (canary), Prometheus/Grafana/Alertmanager, Velero DR, đo DORA |

## Phiên bản — luôn dùng bản mới nhất

Cài bằng lệnh tự lấy bản mới nhất khi có thể. Phiên bản tham chiếu tại thời điểm biên soạn (6/2026):

| Thành phần | Bản (6/2026) | Nguồn |
|---|---|---|
| Harbor | v2.14.4 | tự dò GitHub API |
| Elastic Stack (ECK quản lý) | 9.4.2 | manifest ECK |
| ECK operator | 3.4.0 | Helm `elastic/eck-operator` |
| Strimzi (Kafka operator) | mới nhất | Helm `strimzi/strimzi-kafka-operator` |
| Argo CD / Argo Rollouts | mới nhất | manifest/Helm chính thức |
| cert-manager, ingress-nginx, Kyverno, ESO, Velero, kube-prometheus-stack | mới nhất | Helm chính thức |
| Jenkins | LTS mới nhất | Helm `jenkins/jenkins` |

## Quy ước

```bash
export BASE_DOMAIN="example.com"     # tên miền của bạn (harbor.example.com, jenkins.example.com…)
export S3_ENDPOINT="https://s3.example.com"
```

Mỗi mục có phần **"Kiểm tra"** ở cuối — đạt rồi mới đi tiếp.
