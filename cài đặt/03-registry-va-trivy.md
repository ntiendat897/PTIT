# 03 — Harbor (registry nội bộ) + Trivy (Ubuntu 24.04)

Harbor chạy trên **một server riêng** (`registry.vnpt.vn`), **tích hợp Trivy** để tự động quét lỗ hổng khi push image. TLS bằng **chứng chỉ CA nội bộ VNPT** (không dùng HTTP). Dữ liệu lưu ngay trên server (PostgreSQL/Redis/registry đi kèm), có sao lưu.

> Server đã làm cứng theo file 01. Mở cổng 443 giới hạn theo subnet nội bộ.

```bash
export HARBOR_HOST="registry.vnpt.vn"
export HARBOR_PROJECT="myvnpt"
```

---

## 1. Chuẩn bị Docker + chứng chỉ TLS

```bash
# Docker Engine (noble) — như file 02 mục 2
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
# Chứng chỉ CA nội bộ cấp cho registry.vnpt.vn
sudo mkdir -p /data/cert
sudo cp registry.vnpt.vn.crt /data/cert/harbor.crt
sudo cp registry.vnpt.vn.key /data/cert/harbor.key
```
> Dùng chứng chỉ do **CA nội bộ VNPT** cấp cho `registry.vnpt.vn`. Mọi máy pull/push phải tin CA này (mục 6).

---

## 2. Tải Harbor (bản mới nhất)

```bash
cd ~
HARBOR_VER=$(curl -s https://api.github.com/repos/goharbor/harbor/releases/latest | grep -oP '"tag_name": "\K[^"]+')
echo "Harbor: $HARBOR_VER"
wget https://github.com/goharbor/harbor/releases/download/${HARBOR_VER}/harbor-online-installer-${HARBOR_VER}.tgz
tar xzf harbor-online-installer-${HARBOR_VER}.tgz && cd harbor
cp harbor.yml.tmpl harbor.yml
```

---

## 3. Cấu hình `harbor.yml`

```yaml
hostname: registry.vnpt.vn
http:
  port: 80                      # chỉ redirect sang https
https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key
harbor_admin_password: "ĐẶT_MẬT_KHẨU_MẠNH"
data_volume: /data             # nên là ổ đĩa riêng, mount noatime
# (mặc định) PostgreSQL/Redis/registry chạy kèm — giữ nguyên cho mô hình 1 server
```

---

## 4. Cài đặt kèm Trivy

```bash
sudo ./install.sh --with-trivy
sudo docker compose ps          # các container Up/healthy
curl -kI https://registry.vnpt.vn
```
Mở `https://registry.vnpt.vn`, đăng nhập `admin`.

> Quản lý vòng đời (trong `~/harbor`): `sudo docker compose down|up -d`; sau khi sửa `harbor.yml`: `sudo ./prepare && sudo docker compose up -d`.

---

## 5. Cấu hình bảo mật & quét tự động

1. **Administration → Authentication**: chuyển **LDAP/OIDC** trỏ tới hệ thống tài khoản VNPT; tắt tự đăng ký.
2. **Projects → New Project** `myvnpt` (Private). Vào **Configuration**:
   - Bật **"Automatically scan images on push"** (Trivy tự quét).
   - Bật **"Prevent vulnerable images from running"**, ngưỡng `High`.
   - Đặt **quota** dung lượng; bật **tag retention/immutability**.
3. **Vulnerability → Update** CSDL Trivy định kỳ (hoặc bật tự cập nhật).

---

## 6. Robot account cho CI & tin tưởng CA

**Project `myvnpt` → Robot Accounts → New**: chỉ `push`+`pull` trên project. Lưu token vào credential `jenkins.devops` (file 02).

Cho Docker (server Jenkins/agent) và k8s tin CA nội bộ (TLS thật, **không** dùng insecure):
```bash
# Trên server build
sudo mkdir -p /etc/docker/certs.d/registry.vnpt.vn
sudo cp vnpt-ca.crt /etc/docker/certs.d/registry.vnpt.vn/ca.crt
sudo systemctl restart docker
# Trên node K8s (containerd): khai báo CA cho registry
sudo tee /etc/containerd/certs.d/registry.vnpt.vn/hosts.toml >/dev/null <<EOF
server = "https://registry.vnpt.vn"
[host."https://registry.vnpt.vn"]
  ca = "/etc/ssl/vnpt-ca.crt"
EOF
sudo cp vnpt-ca.crt /etc/ssl/vnpt-ca.crt && sudo systemctl restart containerd
```

Đẩy thử:
```bash
docker login registry.vnpt.vn -u admin
docker pull alpine:latest && docker tag alpine:latest registry.vnpt.vn/myvnpt/test:1 && docker push registry.vnpt.vn/myvnpt/test:1
```

---

## 7. Sao lưu Harbor (DR)

Dữ liệu nằm trong `/data` (DB + layer image). Sao lưu định kỳ:
```bash
cd ~/harbor && sudo docker compose down
sudo tar czf /backup/harbor-$(date +%F).tgz /data ~/harbor/harbor.yml
sudo docker compose up -d
# Đồng bộ ra máy/ổ backup nội bộ
rsync -a /backup/ backup-host:/harbor-backups/
```
Đặt cron hằng đêm và **kiểm thử khôi phục** định kỳ. Gia hạn chứng chỉ trước khi hết hạn.

**Kiểm tra:** push image → Harbor tự quét Trivy và hiển thị CVE; image vượt ngưỡng High bị chặn pull.

---

## Tổng kết file 03
Harbor chạy nội bộ với TLS CA nội bộ, Trivy tự quét, robot account, chính sách chặn lỗ hổng và backup. Sang `04-kafka-elk-cluster.md`.
