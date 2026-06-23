# 05 — Pipeline CI (Maven) + Helm Chart + GitOps với Argo CD

Mô hình thật: **hai kho Git** — kho mã nguồn ứng dụng và kho **`devsecops`** chứa Helm Chart. Jenkins (CI) build/push image rồi **cập nhật `appVersion` trong Helm Chart** ở kho `devsecops`; **Argo CD** theo dõi kho `devsecops` và triển khai lên Kubernetes.

```
app-repo/                      # mã nguồn + Dockerfile + Jenkinsfile
devsecops/                     # các Helm Chart theo môi trường (nguồn sự thật của Argo CD)
└── myvnpt/stage-api-myvnpt-config/{Chart.yaml, values.yaml, templates/}
```

---

## 1. Dockerfile (ứng dụng Java, chạy non-root)

```dockerfile
FROM eclipse-temurin:8-jre        # hoặc 17/21 theo ứng dụng
WORKDIR /app
COPY target/*.jar app.jar
RUN useradd -r -u 1001 appuser
USER appuser
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```
> Mã nguồn được build bằng Maven trong pipeline (image `maven:3.9.9-eclipse-temurin-8`), sau đó đóng gói jar vào image chạy non-root.

---

## 2. Helm Chart trong kho devsecops

```bash
# Trên server agent, tại /opt/jenkins/workspace/devsecops
helm create stage-api-myvnpt-config
```
Sửa `values.yaml`:
```yaml
image:
  repository: registry.vnpt.vn/myvnpt/api-myvnpt-config   # khớp image push từ pipeline
  pullPolicy: IfNotPresent
imagePullSecrets:
  - name: regcred                      # secret kéo image từ Harbor (mục 5)
replicaCount: 2
service:
  type: ClusterIP
  port: 80
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
```
`Chart.yaml` có dòng `appVersion` — chính là **tag image** mà pipeline sẽ cập nhật:
```yaml
appVersion: "stage-v260408-01"
```
Commit chart lên kho `devsecops`.

---

## 3. Jenkinsfile (CI) — đúng luồng VNPT

Đặt trong kho mã nguồn ứng dụng. Kích hoạt khi **đánh tag**; build Maven; SonarQube; build & push image; cập nhật `appVersion` ở kho devsecops; thông báo Telegram.

```groovy
def appSourceRepo = 'https://gitlab.myvnpt.com.vn/gds/myvnpt/api_myvnpt_config.git'
def appSourceBranch = 'master'
def devopsRepo = 'https://gitlab.myvnpt.com.vn/devsecops/devsecops.git'
def devopsBranch = 'main'
def devopsFolder = "/opt/jenkins/workspace/devsecops"
def helmChartFile = "myvnpt/stage-api-myvnpt-config/Chart.yaml"

pipeline {
  agent { label 'agent01' }
  environment {
    DOCKER_REGISTRY = 'https://registry.vnpt.vn'
    DOCKER_IMAGE    = "registry.vnpt.vn/myvnpt/api-myvnpt-config"
  }
  stages {
    stage('Clone source code') {
      steps {
        git branch: appSourceBranch, credentialsId: 'jenkins.devops', url: appSourceRepo
        script { env.GIT_TAG = sh(returnStdout: true, script: "git tag --sort version:refname | grep stage-v | tail -1").trim() }
      }
    }
    stage('Build') {
      steps {
        sh '''docker run --rm -v $PWD:/app -w /app maven:3.9.9-eclipse-temurin-8 mvn clean package -DskipTests'''
      }
    }
    stage('SonarQube analysis') {
      tools { jdk 'jdk17' }
      environment { scannerHome = tool 'SonarScanner' }
      steps { withSonarQubeEnv('GDS Sonarqube') { sh "${scannerHome}/bin/sonar-scanner" } }
    }
    stage('Build & Publish image') {
      steps {
        script {
          def img = docker.build("${DOCKER_IMAGE}:${env.GIT_TAG}", ".")
          docker.withRegistry(DOCKER_REGISTRY, 'jenkins.devops') { img.push() }   // Harbor tự quét Trivy khi push
          sh "docker rmi ${DOCKER_IMAGE}:${env.GIT_TAG} || true"
        }
      }
    }
    stage('Update Helm Chart (devsecops)') {
      steps {
        dir(devopsFolder) {
          git branch: devopsBranch, credentialsId: 'jenkins.devops', url: devopsRepo
          withCredentials([gitUsernamePassword(credentialsId: 'jenkins.devops', gitToolName: 'git-tool')]) {
            sh """sed -i 's|appVersion: .*|appVersion: \\"${env.GIT_TAG}\\"|' ${helmChartFile}"""
            sh "git commit -am 'Update appVersion to ${env.GIT_TAG}' && git push ${devopsRepo}"
          }
        }
      }
    }
  }
  post {
    success { telegram("🚀 Build Success — ${DOCKER_IMAGE}:${env.GIT_TAG}") }
    failure { telegram("🔥 Build Failed — ${DOCKER_IMAGE}:${env.GIT_TAG}") }
  }
}
def telegram(msg) {
  withCredentials([string(credentialsId: 'telegram-token', variable: 'TG')]) {
    sh """curl -s -X POST https://api.telegram.org/bot${TG}/sendMessage -d chat_id=<CHAT_ID> -d message_thread_id=<THREAD_ID> -d text="${msg}" || true"""
  }
}
```
> Bảo mật: token Telegram/registry lấy từ **credential Jenkins** (file 02), không ghi phẳng trong Jenkinsfile.

---

## 4. Cài Argo CD (trên cụm K8s) và Application

```bash
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm install argocd argo/argo-cd -n argocd \
  --set server.ingress.enabled=true --set server.ingress.ingressClassName=nginx \
  --set server.ingress.hostname=cd.vnpt.vn \
  --set configs.params."server\.insecure"=false
# Khai báo kho devsecops (private) cho Argo CD
argocd repo add https://gitlab.myvnpt.com.vn/devsecops/devsecops.git --username <user> --password <token>
```
`application.yaml` — Argo CD theo dõi Helm Chart trong kho devsecops:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: api-myvnpt-config-stage, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://gitlab.myvnpt.com.vn/devsecops/devsecops.git
    targetRevision: main
    path: myvnpt/stage-api-myvnpt-config
  destination: { server: https://kubernetes.default.svc, namespace: app }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ CreateNamespace=true ]
```
```bash
kubectl apply -f application.yaml
```

---

## 5. Secret kéo image từ Harbor

```bash
kubectl create secret docker-registry regcred -n app \
  --docker-server=registry.vnpt.vn \
  --docker-username='<robot-account>' --docker-password='<robot-token>'
```
Tham chiếu `imagePullSecrets: [{name: regcred}]` trong values.yaml (mục 2).

---

## 6. Vận hành GitOps & rollback

- Phát hành mới: **đánh tag** trên GitLab → pipeline tự cập nhật `appVersion` → Argo CD đồng bộ.
- Rollback: Argo CD UI (`cd.vnpt.vn`) → chọn ứng dụng → **HISTORY AND ROLLBACK** → chọn revision ổn định.

**Kiểm tra:** đánh một tag thử → pipeline chạy đủ stage → Argo CD `Synced/Healthy` → pod chạy image đúng tag.

---

## Tổng kết file 05
Pipeline CI Maven → Harbor (Trivy) → cập nhật Helm Chart ở devsecops → Argo CD triển khai, thông báo Telegram. Sang `06-observability-dr.md`.
