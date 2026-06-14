# 03 — Harbor (registry) & Trivy trên một server

Mô hình: cài **toàn bộ Harbor + Trivy trên một server duy nhất** bằng bộ cài chính thức (Docker Compose), dùng PostgreSQL/Redis/lưu trữ **đi kèm trong gói** (không tách ra ngoài). Tuy gói gọn trên một máy, vẫn giữ chuẩn production: **HTTPS bằng chứng chỉ thật, đổi mật khẩu admin, robot account least-privilege, gating lỗ hổng, yêu cầu ảnh đã ký, và sao lưu**.

```bash
export HARBOR_HOST="harbor.example.com"   # tên miền trỏ về server này
export HARBOR_PROJECT="myapp"
```

> Yêu cầu server: 4 vCPU, 8 GB RAM, ổ đĩa rộng cho image (vd 100 GB+), Ubuntu 22.04, đã cài Docker + Docker Compose (file 01). Mở cổng 80/443.

---

## 1. Chuẩn bị chứng chỉ TLS (bắt buộc — không dùng HTTP)

Chọn một trong hai cách.

### Cách A — Let's Encrypt (server có tên miền ra Internet)
```bash
sudo apt-get install -y certbot
# Lấy chứng chỉ (dừng dịch vụ đang chiếm cổng 80 nếu có)
sudo certbot certonly --standalone -d $HARBOR_HOST
# Chứng chỉ nằm tại:
#   /etc/letsencrypt/live/$HARBOR_HOST/fullchain.pem
#   /etc/letsencrypt/live/$HARBOR_HOST/privkey.pem
```

### Cách B — CA nội bộ công ty / self-signed (mạng nội bộ)
```bash
sudo mkdir -p /data/cert
# Nếu công ty đã có CA: xin cấp cert cho $HARBOR_HOST rồi đặt vào /data/cert/
# Hoặc tự tạo (nội bộ):
openssl req -x509 -newkey rsa:4096 -nodes -days 3650 \
  -keyout /data/cert/harbor.key -out /data/cert/harbor.crt \
  -subj "/CN=$HARBOR_HOST" -addext "subjectAltName=DNS:$HARBOR_HOST"
```
> Dùng CA nội bộ thì phải phân phối **CA gốc** tới mọi máy pull/push (xem Mục 6).

---

## 2. Tải bộ cài Harbor (bản mới nhất)

```bash
cd ~
HARBOR_VER=$(curl -s https://api.github.com/repos/goharbor/harbor/releases/latest | grep -oP '"tag_name": "\K[^"]+')
echo "Harbor moi nhat: $HARBOR_VER"     # tại thời điểm viết là v2.14.4
wget https://github.com/goharbor/harbor/releases/download/${HARBOR_VER}/harbor-online-installer-${HARBOR_VER}.tgz
tar xzvf harbor-online-installer-${HARBOR_VER}.tgz
cd harbor
cp harbor.yml.tmpl harbor.yml
```

---

## 3. Cấu hình `harbor.yml` (HTTPS, một server)

Sửa các mục sau trong `harbor.yml`:

```yaml
hostname: harbor.example.com           # = $HARBOR_HOST

http:
  port: 80                             # chỉ để redirect sang https

https:
  port: 443
  # Cách A (Let's Encrypt):
  certificate: /etc/letsencrypt/live/harbor.example.com/fullchain.pem
  private_key: /etc/letsencrypt/live/harbor.example.com/privkey.pem
  # Cách B (CA nội bộ): trỏ tới /data/cert/harbor.crt và /data/cert/harbor.key

harbor_admin_password: "DAT_MAT_KHAU_MANH_O_DAY"   # đổi ngay, không để mặc định

# Lưu trữ ngay trên server này (không tách S3)
data_volume: /data

# (mặc định) PostgreSQL & Redis chạy kèm trong gói — giữ nguyên, không cấu hình external
```

> Lưu ý: để mặc định nghĩa là Harbor tự chạy database + redis + registry trong các container trên chính server này — đúng yêu cầu "không tách đi đâu".

---

## 4. Cài đặt kèm Trivy

```bash
sudo ./install.sh --with-trivy
```
Cờ `--with-trivy` bật scanner Trivy ngay trong Harbor (quét ảnh khi push). Khi xong sẽ hiện "✔ ----Harbor has been installed and started successfully----".

**Kiểm tra:**
```bash
sudo docker compose ps               # tất cả container Up/healthy (core, db, redis, registry, trivy-adapter, portal…)
curl -I https://$HARBOR_HOST         # trả về HTTP 200/302 qua HTTPS
```
Mở `https://$HARBOR_HOST`, đăng nhập `admin` / mật khẩu vừa đặt.

> Quản lý vòng đời (trong thư mục `~/harbor`):
> ```bash
> sudo docker compose down            # dừng
> sudo docker compose up -d           # chạy lại
> sudo ./prepare && sudo docker compose up -d   # sau khi sửa harbor.yml hoặc gia hạn cert
> ```

---

## 5. Cấu hình bảo mật trong Harbor

1. (Khuyến nghị) **Administration → Configuration → Authentication**: chuyển **OIDC/LDAP** trỏ tới hệ thống tài khoản công ty; tắt tự đăng ký.
2. **Projects → New Project** tên `myapp` (Private). Vào **Configuration** của project:
   - Bật **"Automatically scan images on push"** (Trivy tự quét).
   - Bật **"Prevent vulnerable images from running"**, ngưỡng `High` → chặn pull ảnh vượt ngưỡng.
   - Bật **Cosign** trong Content Trust để yêu cầu ảnh đã ký.
3. Đặt **quota** dung lượng và **tag retention/immutability** để giữ lịch sử, chống ghi đè tag.

---

## 6. Cho Docker host & k3s tin tưởng Harbor

### Nếu dùng Let's Encrypt (chứng chỉ công khai, đã tin sẵn)
Không cần thêm gì — Docker và k3s tin chứng chỉ ngay.

### Nếu dùng CA nội bộ / self-signed
Phân phối CA tới mọi máy pull/push (không dùng `insecure-registries`):

```bash
# Trên máy chạy Docker (vd server build/Jenkins)
sudo mkdir -p /etc/docker/certs.d/$HARBOR_HOST
sudo cp ca.crt /etc/docker/certs.d/$HARBOR_HOST/ca.crt
sudo systemctl restart docker

# Trên node k3s: thêm CA vào registries.yaml (dùng TLS, KHÔNG insecure)
sudo tee /etc/rancher/k3s/registries.yaml > /dev/null <<EOF
configs:
  "$HARBOR_HOST":
    tls:
      ca_file: /etc/ssl/harbor-ca.crt
EOF
sudo cp ca.crt /etc/ssl/harbor-ca.crt
sudo systemctl restart k3s
```

Đăng nhập và đẩy thử:
```bash
docker login $HARBOR_HOST -u admin
docker pull alpine:latest
docker tag alpine:latest $HARBOR_HOST/$HARBOR_PROJECT/myapp:test
docker push $HARBOR_HOST/$HARBOR_PROJECT/myapp:test
```
Vào project `myapp` → **Repositories**, thấy `myapp` cùng kết quả quét CVE.

---

## 7. Robot account cho CI (least privilege)

Không dùng admin cho pipeline:
- **Project `myapp` → Robot Accounts → New Robot Account**, chỉ cấp `push` + `pull` trên project này.
- Lưu token an toàn; trong Jenkins tạo credential từ token này (không dùng tài khoản admin).

---

## 8. Trivy CLI (cổng chặn trong pipeline)

Harbor tự quét sau khi push; trong pipeline nên quét **trước khi push** để chặn sớm. Cài Trivy CLI trên server build (hoặc dùng container `aquasec/trivy`):

```bash
sudo apt-get install -y wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy
trivy --version
```

Dùng làm cổng chặn:
```bash
trivy image --severity HIGH,CRITICAL --exit-code 0 $HARBOR_HOST/$HARBOR_PROJECT/myapp:$TAG   # báo cáo
trivy image --severity CRITICAL --exit-code 1 $HARBOR_HOST/$HARBOR_PROJECT/myapp:$TAG          # FAIL build nếu có CRITICAL
```

| Lớp quét | Khi nào | Vai trò |
|---|---|---|
| Trivy CLI (pipeline) | Trước khi push | Chặn sớm, không cho ảnh lỗi lên registry |
| Trivy trong Harbor | Sau khi push + định kỳ | Lưu lịch sử CVE, chặn pull theo chính sách |

---

## 9. Sao lưu Harbor (trên cùng server + bản offsite)

Vì mọi dữ liệu nằm trên một server, backup càng quan trọng. Dữ liệu cần sao lưu nằm trong `data_volume` (`/data`): database (metadata) và layer image.

```bash
# Dừng để snapshot nhất quán (hoặc dùng cơ chế dump DB của Harbor)
cd ~/harbor && sudo docker compose down
sudo tar czf /backup/harbor-$(date +%F).tgz /data ~/harbor/harbor.yml
sudo docker compose up -d
# Sao thêm một bản ra máy/ổ khác (offsite) để phòng hỏng server
rsync -a /backup/ user@backup-host:/harbor-backups/
```
Đặt lịch (cron) chạy hằng đêm và **kiểm thử khôi phục định kỳ**. Nếu dùng Let's Encrypt, đặt cron `certbot renew` + `./prepare && docker compose up -d` để gia hạn cert.

---

## Liên hệ với file 05

- Tên image: `harbor.example.com/myapp/<app>`.
- Pull secret cho cụm: tạo `regcred` từ robot account; Deployment/Helm tham chiếu `imagePullSecrets`.
- Credential Jenkins: dùng token robot account (ID `harbor`), không dùng admin.

---

## Tổng kết file 03

Harbor + Trivy chạy gọn trên một server với HTTPS thật, gating CVE, yêu cầu ảnh đã ký, robot account và backup. Sang `04-kafka-elk-cluster.md`.
