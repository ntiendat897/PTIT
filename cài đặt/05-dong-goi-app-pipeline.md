# 05 — Đóng gói ứng dụng, CI và GitOps (production)

Mô hình production tách hai vai trò rõ ràng:
- **Jenkins (CI)**: build rootless → test → quét → ký → push Harbor → **commit image tag mới vào repo manifests**.
- **Argo CD (CD/GitOps)**: phát hiện thay đổi ở repo manifests → đồng bộ vào cụm.

Hai kho Git tách biệt:
```
app-repo/        # mã nguồn + Dockerfile + Jenkinsfile
manifests-repo/  # Helm chart / Kustomize cho môi trường (staging, prod) — nguồn sự thật của Argo CD
```

---

## 1. Dockerfile hardened

Nguyên tắc: multi-stage, base tối giản, **chạy non-root**, không chứa công cụ build trong image cuối, pin theo digest khi có thể.

Ví dụ Node.js (đổi cho ngôn ngữ của bạn — Node 22 / JDK 21 / Python 3.13 LTS):
```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build --if-present

FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app .
# Chạy non-root
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 3000
CMD ["node", "server.js"]
```
Quét cấu hình Dockerfile: `trivy config .`

---

## 2. Helm chart production (trong manifests-repo)

`values-prod.yaml` — các điểm bắt buộc production:
```yaml
replicaCount: 3
image:
  repository: harbor.example.com/myapp/myapp
  # dùng digest thay vì tag để bất biến (Jenkins cập nhật digest)
  digest: ""
  pullPolicy: IfNotPresent
imagePullSecrets: [ { name: regcred } ]

resources:
  requests: { cpu: "250m", memory: 256Mi }
  limits:   { cpu: "1", memory: 512Mi }

autoscaling:                 # HPA
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2

securityContext:             # chạy non-root, read-only rootfs
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }

probes:
  readiness: { path: /healthz, port: 3000 }
  liveness:  { path: /healthz, port: 3000 }
  startup:   { path: /healthz, port: 3000 }

affinity:                    # trải pod ra nhiều node
  podAntiAffinity: preferredDuringScheduling

networkPolicy:
  enabled: true
```
Template Deployment cần khai báo đủ 3 probe, securityContext, topologySpreadConstraints/anti-affinity, và `image: {{ .repository }}@{{ .digest }}`.

---

## 3. Jenkinsfile (CI) — build, scan, sign, push, bump manifest

```groovy
pipeline {
  agent {
    kubernetes {
      yaml libraryResource('agent-pod.yaml')   // pod kaniko+trivy+cosign (file 02)
    }
  }
  environment {
    HARBOR  = "harbor.example.com"
    PROJECT = "myapp"
    IMAGE   = "harbor.example.com/myapp/myapp"
  }
  stages {
    stage('Unit test') {
      steps { container('kaniko') { sh 'echo "chạy test trong build context / hoặc stage riêng"' } }
    }
    stage('Build & Push (Kaniko, rootless)') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor --context `pwd` --dockerfile Dockerfile \
              --destination $IMAGE:$GIT_COMMIT \
              --digest-file /workspace/digest --no-push=false
          '''
          script { env.DIGEST = readFile('/workspace/digest').trim() }
        }
      }
    }
    stage('Security scan (Trivy gate)') {
      steps {
        container('trivy') {
          sh 'trivy image --severity HIGH,CRITICAL --exit-code 0 $IMAGE@$DIGEST'
          sh 'trivy image --severity CRITICAL --exit-code 1 $IMAGE@$DIGEST'   // fail nếu CRITICAL
          sh 'trivy image --format cyclonedx -o sbom.json $IMAGE@$DIGEST'
        }
      }
    }
    stage('Sign (Cosign)') {
      steps {
        container('cosign') {
          withCredentials([string(credentialsId: 'cosign-password', variable: 'COSIGN_PASSWORD')]) {
            sh 'cosign sign --key env://COSIGN_KEY $IMAGE@$DIGEST'
            sh 'cosign attach sbom --sbom sbom.json $IMAGE@$DIGEST'
          }
        }
      }
    }
    stage('Bump manifest (GitOps)') {
      steps {
        withCredentials([string(credentialsId: 'git-token', variable: 'GIT_TOKEN')]) {
          sh '''
            git clone https://$GIT_TOKEN@git.example.com/org/manifests-repo.git
            cd manifests-repo
            yq -i ".image.digest = \\"$DIGEST\\"" envs/prod/values.yaml
            git commit -am "ci: myapp $GIT_COMMIT -> $DIGEST"
            git push
          '''
        }
      }
    }
  }
}
```
Jenkins **dừng tại đây** — không deploy. Việc còn lại do Argo CD.

---

## 4. Argo CD (GitOps CD)

### 4.1 Cài Argo CD
```bash
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd \
  --set server.ingress.enabled=true \
  --set server.ingress.ingressClassName=nginx \
  --set server.ingress.hostname=argocd.example.com \
  --set 'server.ingress.annotations.cert-manager\.io/cluster-issuer=letsencrypt-prod'
```

### 4.2 Application trỏ tới manifests-repo — `app-prod.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: myapp-prod, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://git.example.com/org/manifests-repo.git
    targetRevision: main
    path: envs/prod
    helm: { valueFiles: [values.yaml] }
  destination: { server: https://kubernetes.default.svc, namespace: app }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=false]
```
```bash
kubectl apply -f app-prod.yaml
```
Từ giờ: Jenkins commit digest mới → Argo CD tự đồng bộ → cụm cập nhật. `selfHeal` đưa cụm về đúng trạng thái Git nếu ai đó sửa tay.

---

## 5. Kyverno — chỉ chạy image đã ký & từ Harbor

```bash
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```
`verify-images.yaml`:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: verify-image-signature }
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-cosign-signature
      match: { any: [ { resources: { kinds: [Pod], namespaces: [app] } } ] }
      verifyImages:
        - imageReferences: ["harbor.example.com/myapp/*"]
          attestors:
            - entries:
                - keys: { publicKeys: |-
                    -----BEGIN PUBLIC KEY-----
                    <cosign-public-key>
                    -----END PUBLIC KEY----- }
```
Thêm policy chặn image ngoài Harbor, ép `runAsNonRoot`, cấm `latest` tag. Đây là chốt chặn supply-chain trong cụm.

---

## Tổng kết file 05

Đã có: Dockerfile non-root, Helm chart production (HPA/PDB/probes/securityContext/NetworkPolicy), CI Jenkins build–scan–sign–push–bump, **Argo CD** đồng bộ GitOps, và **Kyverno** chỉ cho chạy image đã ký. Sang `06-rollout-observability-dr.md`.
