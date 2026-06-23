# 01 — Làm cứng Ubuntu 24.04, JUMP server, Kubernetes & Rancher

File này gồm ba phần: (A) làm cứng (hardening) áp dụng cho **mọi server Ubuntu 24.04**; (B) thiết lập **JUMP server (bastion)**; (C) dựng **cụm Kubernetes** và **Rancher**.

---

## A. Làm cứng cơ sở cho MỌI server (Ubuntu 24.04)

Thực hiện trên từng server ngay sau khi cài hệ điều hành.

### A.1 Cập nhật và gói cơ bản
```bash
sudo apt update && sudo apt -y full-upgrade
sudo apt install -y vim curl wget git gnupg ca-certificates ufw fail2ban \
  unattended-upgrades chrony auditd apparmor-utils
sudo timedatectl set-timezone Asia/Ho_Chi_Minh
```

### A.2 Người dùng quản trị & vô hiệu hóa root
```bash
sudo adduser $ADMIN_USER
sudo usermod -aG sudo $ADMIN_USER
sudo mkdir -p /home/$ADMIN_USER/.ssh && sudo chmod 700 /home/$ADMIN_USER/.ssh
# Dán public key của quản trị viên vào authorized_keys (chỉ đăng nhập bằng khóa)
sudo vim /home/$ADMIN_USER/.ssh/authorized_keys
sudo chmod 600 /home/$ADMIN_USER/.ssh/authorized_keys
sudo chown -R $ADMIN_USER:$ADMIN_USER /home/$ADMIN_USER/.ssh
```

### A.3 Làm cứng SSH
Sửa `/etc/ssh/sshd_config` (hoặc tạo `/etc/ssh/sshd_config.d/99-hardening.conf`):
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
X11Forwarding no
MaxAuthTries 3
AllowUsers devops
ClientAliveInterval 300
ClientAliveCountMax 2
```
```bash
sudo systemctl restart ssh
```

### A.4 Tường lửa (UFW) — mặc định chặn
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
# Trên các server thường: CHỈ cho JUMP server SSH vào
sudo ufw allow from $JUMP_IP to any port 22 proto tcp
sudo ufw --force enable
```
> Trên JUMP server thì cho phép SSH từ dải mạng quản trị (xem phần B). Các cổng dịch vụ (Harbor 443, Kafka 9092/9093… ) sẽ mở giới hạn theo subnet ở từng file tương ứng.

### A.5 fail2ban (chống dò mật khẩu)
```bash
sudo tee /etc/fail2ban/jail.local > /dev/null <<'EOF'
[sshd]
enabled = true
bantime = 1h
findtime = 10m
maxretry = 5
EOF
sudo systemctl enable --now fail2ban
```

### A.6 Cập nhật bảo mật tự động
```bash
sudo dpkg-reconfigure -plow unattended-upgrades   # chọn Yes
# hoặc bật thủ công:
echo 'APT::Periodic::Unattended-Upgrade "1";' | sudo tee /etc/apt/apt.conf.d/20auto-upgrades
```

### A.7 Tối ưu kernel & giới hạn (sysctl/ulimit)
```bash
sudo tee /etc/sysctl.d/99-prod.conf > /dev/null <<'EOF'
vm.swappiness = 1
vm.max_map_count = 262144
fs.file-max = 2097152
net.core.somaxconn = 1024
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
EOF
sudo sysctl --system
sudo tee /etc/security/limits.d/99-prod.conf > /dev/null <<'EOF'
* soft nofile 100000
* hard nofile 100000
EOF
```

### A.8 Đồng bộ thời gian
```bash
sudo systemctl enable --now chrony
timedatectl   # "System clock synchronized: yes"
```

**Kiểm tra (mỗi server):** `ssh` chỉ vào được bằng khóa từ JUMP; `sudo ufw status` đúng quy tắc; `systemctl is-active fail2ban chrony` đều active.

---

## B. JUMP server (bastion)

JUMP là **điểm vào quản trị duy nhất**: quản trị viên SSH vào JUMP rồi mới SSH tiếp tới các server.

### B.1 Tường lửa JUMP
```bash
# Chỉ cho dải mạng quản trị (VD VPN/LAN quản trị) SSH vào JUMP
sudo ufw allow from 10.0.0.0/24 to any port 22 proto tcp
sudo ufw default deny incoming && sudo ufw --force enable
```

### B.2 MFA cho đăng nhập JUMP (Google Authenticator)
```bash
sudo apt install -y libpam-google-authenticator
# Mỗi quản trị viên chạy 'google-authenticator' để tạo mã
```
Bật trong `/etc/pam.d/sshd`: thêm `auth required pam_google_authenticator.so`; trong sshd_config đặt `KbdInteractiveAuthentication yes` và `AuthenticationMethods publickey,keyboard-interactive`.

### B.3 Ghi log phiên (audit)
```bash
sudo systemctl enable --now auditd
# (Tùy chọn) cài 'tlog' hoặc ghi session để phục vụ kiểm toán quản trị
```

**Kiểm tra:** từ máy quản trị → SSH JUMP (yêu cầu khóa + mã MFA) → từ JUMP SSH được tới các server; các server từ chối SSH trực tiếp ngoài JUMP.

---

## C. Cụm Kubernetes (kubeadm, HA)

Production nên dùng **≥ 3 control-plane + ≥ 2 worker**. Tóm tắt với kubeadm (containerd).

### C.1 Chuẩn bị node K8s
```bash
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab   # K8s yêu cầu tắt swap
sudo modprobe overlay && sudo modprobe br_netfilter
sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<<'overlay
br_netfilter'
sudo tee /etc/sysctl.d/k8s.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
# containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### C.2 Cài kubeadm/kubelet/kubectl
```bash
KVER=v1.31
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KVER/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://pkgs.k8s.io/core:/stable:/$KVER/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl && sudo apt-mark hold kubelet kubeadm kubectl
```

### C.3 Khởi tạo control-plane (HA qua load balancer)
```bash
sudo kubeadm init --control-plane-endpoint "<API_LB_DNS>:6443" \
  --upload-certs --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube && sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
# CNI hỗ trợ NetworkPolicy (Calico)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```
Tham gia thêm control-plane (`kubeadm join ... --control-plane`) và worker (`kubeadm join ...`) theo lệnh kubeadm in ra.

### C.4 etcd snapshot định kỳ (DR)
```bash
# CronJob đẩy snapshot etcd ra lưu trữ nội bộ/S3 (giữ ≥ 14 bản)
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key snapshot save /backup/etcd-$(date +%F).db
```

**Kiểm tra:** `kubectl get nodes` Ready đủ số node; `kubectl get pods -A` (calico, coredns) Running.

---

## D. Rancher (quản lý cụm)

Cài Rancher bằng Helm trên cụm (hoặc cụm quản lý riêng), TLS qua cert-manager + CA nội bộ.
```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add jetstack https://charts.jetstack.io && helm repo add rancher https://releases.rancher.com/server-charts/stable && helm repo update
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set crds.enabled=true
helm install rancher rancher/rancher -n cattle-system --create-namespace \
  --set hostname=rancher.$INTERNAL_DOMAIN --set bootstrapPassword=<đặt-mạnh> \
  --set ingress.tls.source=secret   # dùng chứng chỉ CA nội bộ
```
**Kiểm tra:** truy cập `https://rancher.vnpt.vn`, import/quản lý cụm K8s.

---

## E. Namespaces & RBAC (trên cụm K8s)
```bash
for ns in app argocd logging monitoring; do
  kubectl create namespace $ns
  kubectl label ns $ns pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/warn=restricted
done
```
Áp dụng NetworkPolicy mặc định chặn cho namespace `app` rồi mở theo nhu cầu.

---

## Tổng kết file 01
Mọi server đã được làm cứng, JUMP bastion hoạt động, cụm Kubernetes HA + Rancher sẵn sàng. Sang `02-jenkins.md`.
