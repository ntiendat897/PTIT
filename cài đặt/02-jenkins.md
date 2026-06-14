# 02 — Jenkins CI (production)

Jenkins chỉ đảm nhiệm **CI**: build image **rootless**, test, quét bảo mật, **ký image**, push lên Harbor, rồi cập nhật image tag vào repo manifests. Việc triển khai lên cụm do **Argo CD** lo (xem file 05). Jenkins **không** có quyền `kubectl apply` lên production.

Khác biệt so với bản lab:
- Không mount `docker.sock`, không `chmod 666` — build bằng **Kaniko** (rootless, trong pod).
- Jenkins chạy **trên K8s** qua Helm, dùng **agent động** (mỗi build một pod, tự hủy).
- Truy cập qua **HTTPS/Ingress**; bí mật lấy từ **Vault** qua External Secrets.

---

## 1. Cài Jenkins trên K8s qua Helm

`jenkins-values.yaml`:
```yaml
controller:
  image:
    tag: "lts-jdk21"
  replicaCount: 1
  resources:
    requests: { cpu: "1", memory: 2Gi }
    limits:   { cpu: "2", memory: 4Gi }
  ingress:
    enabled: true
    ingressClassName: nginx
    hostName: jenkins.example.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    tls:
      - secretName: jenkins-tls
        hosts: [ jenkins.example.com ]
  installPlugins:
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - configuration-as-code:latest
    - prometheus:latest          # xuất metrics cho Prometheus
  JCasC:
    defaultConfig: true
  additionalExistingSecrets: []
persistence:
  enabled: true
  storageClass: "<ten-sc>"
  size: 20Gi
serviceAccount:
  create: true
agent:
  enabled: true                  # bật agent động trên K8s
  podName: jenkins-agent
  resources:
    requests: { cpu: "500m", memory: 1Gi }
    limits:   { cpu: "2", memory: 4Gi }
rbac:
  create: true
```

```bash
helm install jenkins jenkins/jenkins -n jenkins -f jenkins-values.yaml
```

**Kiểm tra:**
```bash
kubectl -n jenkins get pods,ingress
# Mật khẩu admin ban đầu:
kubectl -n jenkins exec svc/jenkins -c jenkins -- \
  cat /run/secrets/additional/chart-admin-password 2>/dev/null || \
kubectl -n jenkins get secret jenkins -o go-template='{{.data.jenkins-admin-password | base64decode}}'
```
Truy cập `https://jenkins.example.com` (chứng chỉ do cert-manager cấp).

> Khuyến nghị production: thay đăng nhập admin nội bộ bằng **OIDC/LDAP** (qua plugin `oic-auth`/`ldap`), cấu hình bằng **JCasC** để toàn bộ Jenkins là code.

---

## 2. Bí mật từ Vault (không để phẳng)

Tạo `ExternalSecret` kéo thông tin từ Vault thành Secret K8s mà pipeline dùng:

`jenkins-externalsecrets.yaml`:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: ci-credentials, namespace: jenkins }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: vault-backend, kind: ClusterSecretStore }
  target: { name: ci-credentials }
  data:
    - secretKey: harbor-robot-user
      remoteRef: { key: ci/harbor, property: username }
    - secretKey: harbor-robot-pass
      remoteRef: { key: ci/harbor, property: password }
    - secretKey: cosign-key
      remoteRef: { key: ci/cosign, property: key }
    - secretKey: cosign-password
      remoteRef: { key: ci/cosign, property: password }
    - secretKey: git-token
      remoteRef: { key: ci/git, property: token }
```
```bash
kubectl apply -f jenkins-externalsecrets.yaml
kubectl -n jenkins get secret ci-credentials
```

---

## 3. Build image rootless bằng Kaniko (pod agent)

Định nghĩa pod agent có 3 container: `kaniko` (build+push), `trivy` (scan), `cosign` (ký). Đưa vào pipeline ở file 05; phần khai báo pod template (Jenkins Kubernetes plugin):

```groovy
// Khai báo trong Jenkinsfile (xem file 05) — agent pod nhiều container
podTemplate(yaml: '''
  apiVersion: v1
  kind: Pod
  spec:
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
    containers:
      - name: kaniko
        image: gcr.io/kaniko-project/executor:latest
        command: ["sleep"]
        args: ["infinity"]
        volumeMounts:
          - name: harbor-auth
            mountPath: /kaniko/.docker
      - name: trivy
        image: aquasec/trivy:latest
        command: ["sleep"]
        args: ["infinity"]
      - name: cosign
        image: gcr.io/projectsigstore/cosign:latest
        command: ["sleep"]
        args: ["infinity"]
    volumes:
      - name: harbor-auth
        secret:
          secretName: ci-credentials
          items:
            - key: harbor-dockerconfig
              path: config.json
''') { /* các stage build/scan/sign/push ở file 05 */ }
```

Vì sao Kaniko: build image trong userspace, **không cần Docker daemon, không cần quyền root, không mount socket** — loại bỏ lỗ hổng lớn nhất của bản lab.

---

## 4. Ký image với Cosign (supply-chain security)

Trong pipeline, sau khi push image, ký bằng khóa Cosign lấy từ Vault:
```bash
export COSIGN_PASSWORD="$cosign_password"
cosign sign --key env://COSIGN_KEY $HARBOR/$PROJECT/myapp@$DIGEST
```
Cụm sẽ chỉ chạy image có chữ ký hợp lệ nhờ chính sách **Kyverno** (file 05). Khuyến nghị thêm: sinh **SBOM** (`trivy image --format cyclonedx`) và đính kèm (`cosign attach sbom`).

---

## 5. Webhook & phân quyền

- Webhook GitHub/GitLab trỏ tới `https://jenkins.example.com/github-webhook/` (HTTPS).
- Jenkins dùng **robot account** của Harbor (least privilege, chỉ push project tương ứng) — tạo ở file 03; không dùng admin.
- Job định nghĩa bằng **Pipeline from SCM** (Jenkinsfile trong repo), không cấu hình tay.

**Kiểm tra:**
```bash
# Build thử: pod agent xuất hiện rồi tự hủy
kubectl -n jenkins get pods -w
```

---

## Tổng kết file 02

Jenkins chạy trên K8s, HTTPS, agent động, build **rootless Kaniko**, ký **Cosign**, bí mật từ **Vault**, chỉ làm CI và đẩy thay đổi vào Git. Sang `03-registry-va-trivy.md`.
