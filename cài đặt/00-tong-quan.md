# 00 — Tổng quan triển khai (chuẩn production, Ubuntu 24.04)

Bộ tài liệu hướng dẫn cài đặt **hoàn chỉnh** nền tảng CI/CD nội bộ theo đúng hệ thống thực tế, chạy trên **Ubuntu 24.04 LTS**, tối ưu và siết bảo mật theo chuẩn production.

> Mọi thành phần triển khai và sử dụng **nội bộ (on-premise)**, truy cập qua tên miền nội bộ. Không mở dịch vụ ra Internet. Mọi truy cập quản trị bắt buộc qua **JUMP server**.

## Kiến trúc và phân bổ máy chủ

| Máy chủ / cụm | Vai trò | Ghi chú |
|---|---|---|
| JUMP server | Bastion — cổng quản trị duy nhất | MFA, ghi log phiên |
| Rancher | Quản lý cụm Kubernetes | |
| GitLab (`gitlab.myvnpt.com.vn`) | Kho mã nguồn + kho `devsecops` (Helm Chart) | thường đã có sẵn |
| Jenkins + agent (`jenkins.vnpt.vn`) | CI: build Maven, đăng nhập **LDAP** | server riêng |
| SonarQube (+ PostgreSQL) | Cổng chất lượng mã nguồn | thường đã có sẵn |
| Harbor (`registry.vnpt.vn`) | Registry nội bộ + **Trivy** quét lỗ hổng | server riêng |
| Cụm Kubernetes + Argo CD (`cd.vnpt.vn`) | Vận hành + phân phối liên tục (GitOps) | |
| Cụm Kafka 3 node (KRaft) | Hàng đợi đệm log | file `04b` |
| Cụm Elasticsearch + Kibana | Lưu trữ & trực quan hóa log | |
| Prometheus + Grafana | Giám sát metrics, cảnh báo | |

Luồng CI/CD: **đánh tag trên GitLab → webhook → Jenkins** (clone → build Maven → SonarQube → build & push image lên Harbor → cập nhật `appVersion` trong Helm Chart ở kho `devsecops`) **→ Argo CD đồng bộ lên Kubernetes**. Thông báo qua **Telegram**.

## Nguyên tắc production áp dụng xuyên suốt

- **Bảo mật theo lớp**: mọi server làm cứng (hardening) như nhau (file 01); chỉ JUMP được SSH vào các server; tường lửa mặc định chặn; cập nhật bảo mật tự động.
- **TLS nội bộ**: dùng CA nội bộ của VNPT (hoặc tự ký) cho Harbor, Jenkins, Kibana, Grafana, Argo CD, Kafka, Elasticsearch. Không dùng HTTP.
- **Bí mật**: không để mật khẩu/token phẳng trong file cấu hình hay Git; dùng credential của Jenkins, biến môi trường có quyền hạn chế, hoặc Vault (nâng cao).
- **Least privilege**: mỗi dịch vụ chạy bằng user riêng (non-root); phân quyền RBAC trên K8s; robot account trên Harbor; tài khoản LDAP cho Jenkins.
- **Tối ưu tài nguyên**: tách ổ đĩa dữ liệu (SSD, `noatime`); chỉnh `sysctl`, `ulimit`; đặt heap JVM hợp lý (ES/Kafka/Jenkins/Sonar); để RAM còn lại cho page cache.
- **Sao lưu & DR**: snapshot etcd, Elasticsearch, backup Harbor (DB + dữ liệu), backup cấu hình; kiểm thử khôi phục định kỳ.
- **Quan sát đầy đủ**: log (Kafka–ELK) + metrics (Prometheus–Grafana) + thông báo (Telegram).

## Yêu cầu môi trường

- **Hệ điều hành**: Ubuntu 24.04 LTS (64-bit) trên mọi server.
- **Thời gian**: đồng bộ NTP (chrony) toàn hệ thống.
- **DNS nội bộ**: phân giải các tên miền nội bộ (`*.vnpt.vn`, `gitlab.myvnpt.com.vn`).
- **CA/Chứng chỉ**: CA nội bộ của VNPT cấp chứng chỉ cho từng dịch vụ.
- **Tài nguyên tham khảo**: Jenkins 4 vCPU/8GB; Harbor 4 vCPU/8GB/100GB+; mỗi node Kafka 4 vCPU/16GB; mỗi node ES 8 vCPU/32GB; Kibana 2 vCPU/4GB; node K8s tùy tải.

## Thứ tự thực hiện

| File | Nội dung |
|---|---|
| `01-ha-tang-nen.md` | Làm cứng Ubuntu 24.04 cho mọi server; JUMP bastion; cụm Kubernetes; Rancher |
| `02-jenkins.md` | Jenkins trên server riêng + agent (Maven, LDAP, Docker build), HTTPS |
| `03-registry-va-trivy.md` | Harbor (registry.vnpt.vn) + Trivy, TLS nội bộ |
| `04-kafka-elk-cluster.md` | Elasticsearch cluster + Kibana (giám sát log); Kafka xem `04b` |
| `04b-kafka-cluster-3-server.md` | Cụm Kafka 3 server (KRaft, TLS/SASL/ACL) |
| `05-dong-goi-app-pipeline.md` | Helm Chart trong kho devsecops; Argo CD; Jenkinsfile (Maven) + Telegram |
| `06-observability-dr.md` | Prometheus + Grafana; sao lưu/DR; checklist nghiệm thu |

## Quy ước

```bash
# Đặt sẵn (thay theo môi trường VNPT của bạn)
export JUMP_IP="10.0.0.10"           # IP JUMP server
export INTERNAL_DOMAIN="vnpt.vn"
export ADMIN_USER="devops"           # user quản trị (non-root, sudo)
```

Mỗi mục có phần **"Kiểm tra"** — đạt rồi mới đi tiếp.
