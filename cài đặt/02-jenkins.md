# 02 — Jenkins CI trên server riêng (Ubuntu 24.04)

Jenkins chạy trên **một server riêng** (`jenkins.vnpt.vn`) cùng **agent build**, đăng nhập quản trị qua **LDAP**. Pipeline build ứng dụng Java bằng **Maven** (chạy trong container), phân tích SonarQube, đóng gói và đẩy image lên Harbor, cập nhật Helm Chart ở kho `devsecops`, và thông báo Telegram.

> Server này đã được làm cứng theo file 01. Mở thêm cổng web Jenkins giới hạn theo subnet nội bộ.

---

## 1. Cài Java và Jenkins (LTS)

```bash
# Java 21 (Temurin) — cho Jenkins
sudo mkdir -p /etc/apt/keyrings
wget -qO- https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo gpg --dearmor -o /etc/apt/keyrings/adoptium.gpg
echo "deb [signed-by=/etc/apt/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb noble main" | sudo tee /etc/apt/sources.list.d/adoptium.list
# Kho Jenkins LTS
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc >/dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null
sudo apt update && sudo apt install -y temurin-21-jdk jenkins
sudo systemctl enable --now jenkins
```

**Mật khẩu admin ban đầu:**
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## 2. Công cụ build trên server/agent

Pipeline build Maven **chạy trong container Docker** (`docker run maven...`), nên server agent cần Docker; ngoài ra cần Docker để build và push image lên Harbor.

```bash
# Docker Engine (Ubuntu 24.04 - noble)
sudo install -m0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io
# Cho user jenkins dùng Docker (cân nhắc bảo mật: chỉ trên server build nội bộ)
sudo usermod -aG docker jenkins && sudo systemctl restart jenkins
```
> Lưu ý bảo mật: quyền Docker tương đương root. Server build phải được cô lập, chỉ JUMP truy cập, và chỉ chạy pipeline tin cậy. (Phương án rootless/Kaniko là hướng nâng cao.)

---

## 3. HTTPS cho Jenkins (reverse proxy Nginx + TLS nội bộ)

Không expose Jenkins HTTP. Đặt sau Nginx với chứng chỉ CA nội bộ.
```bash
sudo apt install -y nginx
# Đặt chứng chỉ CA nội bộ: /etc/ssl/vnpt/jenkins.crt và jenkins.key
sudo tee /etc/nginx/sites-available/jenkins >/dev/null <<'EOF'
server {
  listen 443 ssl;
  server_name jenkins.vnpt.vn;
  ssl_certificate     /etc/ssl/vnpt/jenkins.crt;
  ssl_certificate_key /etc/ssl/vnpt/jenkins.key;
  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
  }
}
EOF
sudo ln -s /etc/nginx/sites-available/jenkins /etc/nginx/sites-enabled/ && sudo nginx -t && sudo systemctl reload nginx
# Tường lửa: chỉ cho subnet nội bộ truy cập 443
sudo ufw allow from 10.0.0.0/24 to any port 443 proto tcp
```
Trong Jenkins, đặt Jenkins URL = `https://jenkins.vnpt.vn/`.

---

## 4. Đăng nhập LDAP & phân quyền

1. **Manage Jenkins → Plugins**: cài `LDAP`, `Matrix Authorization Strategy`, `Git`, `Pipeline`, `Docker Pipeline`, `SonarQube Scanner`, `Config File Provider`.
2. **Security → Security Realm = LDAP**: trỏ tới máy chủ LDAP nội bộ (server, root DN, user search base).
3. **Authorization = Matrix/Project-based**: cấp quyền theo nhóm LDAP (admin, dev, viewer). Bỏ "anyone can do anything".

---

## 5. Agent build (nút agent01)

Tạo agent để tách tải build khỏi controller (đúng mô hình `label 'agent01'` trong pipeline).
- **Manage Jenkins → Nodes → New Node**: tên `agent01`, Launch method = **SSH** (Jenkins kết nối tới server agent qua SSH bằng credential khóa).
- Trên server agent: cài Java 21 + Docker (như mục 1–2), tạo thư mục `/opt/jenkins/workspace` (khớp `devopsFolder` trong pipeline), quyền cho user agent.

---

## 6. Credentials (không để bí mật phẳng)

**Manage Jenkins → Credentials → System → Global**, thêm:

| Kind | ID | Dùng cho |
|---|---|---|
| Username/Password (hoặc token) | `jenkins.devops` | clone GitLab + push Harbor + push kho devsecops |
| Secret text | `telegram-token` | token bot Telegram |
| Secret text | `sonar-token` | xác thực SonarQube |

> Token Harbor nên là **robot account** (quyền push/pull đúng project), không dùng admin. Tham chiếu các ID này trong Jenkinsfile (file 05).

---

## 7. Tích hợp SonarQube

**Manage Jenkins → System → SonarQube servers**: thêm server `GDS Sonarqube` (URL nội bộ + `sonar-token`). **Global Tool Configuration**: khai báo `SonarScanner` và JDK (`jdk17`) như pipeline yêu cầu.

---

## 8. Webhook từ GitLab (kích hoạt khi đánh tag)

Trong job pipeline, bật **Build Triggers → GitLab webhook**; sinh **secret token**. Trên GitLab khai báo webhook trỏ tới Jenkins, chọn **Tag push events** + dán secret token (chi tiết ở file 05 và đúng tài liệu CI/CD).

**Kiểm tra:** đăng nhập LDAP được; agent `agent01` online; build thử một job → pod/agent chạy `docker run maven` thành công.

---

## Tổng kết file 02
Jenkins chạy trên server riêng sau HTTPS, đăng nhập LDAP, có agent build Docker/Maven, credentials an toàn, tích hợp SonarQube và webhook GitLab. Sang `03-registry-va-trivy.md`.
