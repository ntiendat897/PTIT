# 01 — Hạ tầng nền & platform services (production)

Giả định bạn đã có (hoặc tự dựng) một **cụm Kubernetes HA nhiều node**. File này tập trung vào: bảo đảm cụm đạt chuẩn HA, rồi cài các **platform service** bắt buộc cho production (ingress, TLS, secrets, storage, backup) trước khi triển khai ứng dụng.

---

## 1. Cụm Kubernetes HA

Yêu cầu tối thiểu cho production:

- **≥ 3 control-plane** (etcd theo số lẻ để đạt quorum) + **≥ 2 worker**.
- Phiên bản **≥ 1.31** (yêu cầu của ECK 3.4 và các operator mới).
- Mỗi node tách vai trò; bật `kubelet` resource reserved; phân tách AZ/rack nếu có.

### Nếu tự quản (kubeadm) — tóm tắt HA
- Dựng control-plane đầu tiên với `--control-plane-endpoint` trỏ tới **load balancer** (HAProxy/keepalived) đứng trước API server.
- `kubeadm join --control-plane` cho 2 control-plane còn lại → etcd 3 node.
- Cài CNI (Calico/Cilium) hỗ trợ **NetworkPolicy** (bắt buộc cho production).

### Nếu là k3s HA
```bash
# Server đầu tiên (embedded etcd)
curl -sfL https://get.k3s.io | sh -s - server --cluster-init \
  --tls-san <API_LB_DNS> --disable traefik --disable servicelb
# Các server còn lại tham gia
curl -sfL https://get.k3s.io | K3S_TOKEN=<token> sh -s - server \
  --server https://<API_LB_DNS>:6443 --tls-san <API_LB_DNS> --disable traefik --disable servicelb
```
> Tắt `traefik`/`servicelb` mặc định để tự cài ingress-nginx + MetalLB/LoadBalancer chuẩn production.

### etcd snapshot (DR)
```bash
# k3s: bật snapshot định kỳ ra S3
curl -sfL https://get.k3s.io | sh -s - server \
  --etcd-s3 --etcd-s3-endpoint $S3_ENDPOINT \
  --etcd-s3-bucket k8s-etcd-backup \
  --etcd-snapshot-schedule-cron "0 */6 * * *" --etcd-snapshot-retention 28
# kubeadm: dùng etcdctl snapshot save theo CronJob, đẩy lên S3
```

**Kiểm tra:**
```bash
kubectl get nodes -o wide          # tất cả Ready, đúng số control-plane/worker
kubectl version                    # server ≥ 1.31
kubectl get --raw='/readyz?verbose'
```

---

## 2. StorageClass bền + snapshot

Production cần CSI driver có hỗ trợ snapshot (ví dụ Ceph RBD, Longhorn, hoặc CSI của cloud).

```bash
kubectl get storageclass          # phải có 1 SC mặc định, RWO, allowVolumeExpansion=true
kubectl get volumesnapshotclass   # phục vụ snapshot PVC
```

Đặt SC mặc định nếu chưa có:
```bash
kubectl patch storageclass <ten-sc> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## 3. Helm và các kho chart

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo add external-secrets https://charts.external-secrets.io
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts   # Velero
helm repo add elastic https://helm.elastic.co
helm repo add strimzi https://strimzi.io/charts/
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 4. Ingress controller (ingress-nginx)

Điểm vào HTTPS cho Harbor/Jenkins/Kibana/Grafana/ArgoCD.

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer \
  --set controller.metrics.enabled=true
```
**Kiểm tra:** `kubectl -n ingress-nginx get svc` → có EXTERNAL-IP. Trỏ DNS các tên miền (`*.${BASE_DOMAIN}`) về IP này.

---

## 5. cert-manager (TLS tự động)

```bash
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace --set crds.enabled=true
```

Tạo `ClusterIssuer` (Let's Encrypt production) — `clusterissuer.yaml`:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: letsencrypt-prod }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: devops@example.com
    privateKeySecretRef: { name: letsencrypt-prod }
    solvers:
      - http01:
          ingress: { class: nginx }
```
```bash
kubectl apply -f clusterissuer.yaml
kubectl get clusterissuer       # READY=True
```
> Nội bộ không ra Internet: thay bằng CA nội bộ (issuer kiểu CA) và phân phối root CA tới client.

---

## 6. Quản lý bí mật: Vault + External Secrets Operator

```bash
# Vault (production: nên dùng cụm Vault sẵn có; dưới đây là dạng có HA + storage)
helm install vault hashicorp/vault -n vault --create-namespace \
  --set "server.ha.enabled=true" --set "server.ha.replicas=3"

# External Secrets Operator: đồng bộ secret từ Vault vào K8s
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```
Sau khi `vault operator init`/`unseal`, tạo `SecretStore` trỏ tới Vault. Từ đây mọi mật khẩu (Harbor, Jenkins, Grafana, Elastic…) lấy qua `ExternalSecret`, **không** ghi phẳng vào Git.

> Phương án nhẹ hơn nhưng vẫn hợp lệ production: **Sealed Secrets** (Bitnami) — mã hóa secret để commit an toàn vào Git, phù hợp GitOps.

---

## 7. Backup cụm: Velero

```bash
helm install velero vmware-tanzu/velero -n velero --create-namespace \
  --set-file credentials.secretContents.cloud=./credentials-velero \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=velero-backup \
  --set configuration.backupStorageLocation[0].config.region=default \
  --set configuration.backupStorageLocation[0].config.s3Url=$S3_ENDPOINT \
  --set "initContainers[0].name=velero-plugin-for-aws" \
  --set "initContainers[0].image=velero/velero-plugin-for-aws:latest"
# Lịch backup hằng ngày
velero schedule create daily --schedule "0 2 * * *" --ttl 720h0m0s
```

---

## 8. Namespaces & RBAC

```bash
for ns in app jenkins registry logging monitoring argocd; do
  kubectl create namespace $ns
  kubectl label namespace $ns pod-security.kubernetes.io/enforce=restricted \
    pod-security.kubernetes.io/warn=restricted
done
```
> Gắn nhãn **Pod Security Admission = restricted** để ép workload chạy non-root, no-privilege theo chuẩn production. (Namespace hệ thống của operator có thể cần `baseline` — điều chỉnh khi cần.)

Áp dụng **NetworkPolicy mặc định chặn hết**, rồi mở theo nhu cầu (ví dụ cho namespace `app`):
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: app }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

---

## Tổng kết file 01

Cụm K8s HA đã sẵn sàng với: StorageClass bền, ingress-nginx, cert-manager (TLS), Vault + ESO (bí mật), Velero (backup), namespaces gắn Pod Security + NetworkPolicy mặc định. Sang `02-jenkins.md`.
