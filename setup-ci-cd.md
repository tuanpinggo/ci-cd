# Hướng dẫn cài đặt Jenkins và Argo CD

Kiến trúc triển khai:

- Server 1: Jenkins controller.
- Server 2: Argo CD staging chạy trong K3s staging.
- Server 3: Argo CD production chạy trong K3s production.
- Jenkins không được cấp kubeconfig và không triển khai trực tiếp vào K3s.

Tài liệu giả định cả ba máy dùng Ubuntu 22.04/24.04. Nếu sử dụng hệ điều hành khác, các lệnh cài Jenkins cần được điều chỉnh.

## 1. Kiểm tra trước khi cài

Server 1 nên có tối thiểu:

- 4 CPU;
- 8 GB RAM;
- 50 GB ổ đĩa trống;
- IP tĩnh;
- kết nối outbound Internet.

Trên Server 2 và Server 3, kiểm tra K3s:

```bash
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
sudo k3s kubectl version
```

Node phải ở trạng thái `Ready`.

Argo CD 3.5 được kiểm thử với Kubernetes 1.33–1.36. Nếu phiên bản K3s thấp hơn 1.33, chưa chạy lệnh cài Argo CD bên dưới; cần chọn phiên bản Argo CD tương thích hoặc nâng K3s trước.

Tham khảo: [Argo CD installation and compatibility](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)

---

## Phần A — Cài Jenkins trên Server 1

### 2. Đặt hostname và cập nhật hệ thống

SSH vào Server 1:

```bash
sudo hostnamectl set-hostname cicd-server
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget ca-certificates fontconfig
```

### 3. Cài Java 21

Jenkins hiện yêu cầu Java 21 trở lên.

```bash
sudo apt install -y openjdk-21-jre
java -version
```

Kết quả cần có dạng:

```text
openjdk version "21..."
```

Tham khảo: [Installing Jenkins on Linux](https://www.jenkins.io/doc/book/installing/linux/)

### 4. Thêm repository Jenkins LTS

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null

sudo apt update
sudo apt install -y jenkins
```

### 5. Khởi động Jenkins

```bash
sudo systemctl enable --now jenkins
sudo systemctl --no-pager --full status jenkins
```

Kiểm tra port:

```bash
sudo ss -lntp | grep 8080
```

Nếu Jenkins không chạy:

```bash
sudo journalctl -u jenkins -n 100 --no-pager
```

### 6. Cấu hình firewall

Không nên public port `8080` ra toàn Internet.

Kiểm tra firewall:

```bash
sudo ufw status
```

Nếu UFW đang hoạt động, chỉ cho phép mạng quản trị, ví dụ `10.10.0.0/24`:

```bash
sudo ufw allow from 10.10.0.0/24 to any port 8080 proto tcp
```

Không bật UFW mới nếu chưa chắc rule SSH đã được cho phép, vì có thể tự khóa kết nối SSH.

### 7. Mở Jenkins lần đầu

Lấy mật khẩu ban đầu:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Truy cập:

```text
http://IP_SERVER_1:8080
```

Thực hiện:

1. Nhập initial password.
2. Chọn `Install suggested plugins`.
3. Tạo tài khoản quản trị mới.
4. Đặt Jenkins URL tạm thời thành `http://IP_SERVER_1:8080/`.

Sau này sẽ cấu hình domain và HTTPS, ví dụ:

```text
https://jenkins.example.com
```

### 8. Cài plugin cần thiết

Vào `Manage Jenkins → Plugins`, kiểm tra hoặc cài:

- Git;
- Pipeline;
- GitHub;
- GitHub Branch Source;
- Credentials Binding;
- SSH Agent;
- Docker Pipeline;
- Configuration as Code.

Không nên cài quá nhiều plugin không dùng đến vì làm tăng rủi ro bảo mật và khó nâng cấp.

### 9. Không chạy build trên controller

Vào:

```text
Manage Jenkins
→ Nodes
→ Built-In Node
→ Configure
→ Number of executors = 0
```

Jenkins controller chỉ điều phối, còn build chạy trên agent riêng. Sau khi đặt `0`, Jenkins chưa thể build cho đến khi cấu hình agent. Đây là trạng thái mong muốn.

Tham khảo: [Using Jenkins agents](https://www.jenkins.io/doc/book/using/using-agents/)

---

## Phần B — Cài Argo CD staging trên Server 2

### 10. Kiểm tra phiên bản Kubernetes

Trên Server 2:

```bash
sudo k3s kubectl version
sudo k3s kubectl get nodes
```

Nếu Kubernetes nằm trong khoảng 1.33–1.36, cài Argo CD 3.5.3.

Tham khảo: [Argo CD v3.5.3](https://github.com/argoproj/argo-cd/releases/tag/v3.5.3)

### 11. Cài Argo CD

```bash
sudo k3s kubectl create namespace argocd

sudo k3s kubectl apply \
  -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.3/manifests/install.yaml
```

Việc pin `v3.5.3` giúp tránh hệ thống tự động nhận phiên bản mới ngoài dự kiến.

Chờ các pod sẵn sàng:

```bash
sudo k3s kubectl wait \
  --for=condition=Ready \
  pods \
  --all \
  -n argocd \
  --timeout=600s
```

Kiểm tra:

```bash
sudo k3s kubectl get pods -n argocd
sudo k3s kubectl get svc -n argocd
```

Các pod chính phải ở trạng thái `Running`, chẳng hạn:

```text
argocd-application-controller
argocd-applicationset-controller
argocd-dex-server
argocd-notifications-controller
argocd-redis
argocd-repo-server
argocd-server
```

Tham khảo: [Argo CD Getting Started](https://argo-cd.readthedocs.io/en/latest/getting_started/)

### 12. Truy cập giao diện an toàn bằng SSH tunnel

Không cần mở NodePort hoặc public Argo CD ở giai đoạn này.

Trên Server 2, chạy và giữ terminal này mở:

```bash
sudo k3s kubectl \
  -n argocd \
  port-forward \
  svc/argocd-server \
  8443:443 \
  --address 127.0.0.1
```

Trên máy cá nhân, mở terminal khác:

```bash
ssh -L 8443:127.0.0.1:8443 ubuntu@IP_SERVER_2
```

Truy cập:

```text
https://localhost:8443
```

Trình duyệt có thể cảnh báo chứng chỉ self-signed; đây là truy cập tạm thời.

### 13. Lấy mật khẩu Argo CD ban đầu

Trên Server 2:

```bash
sudo k3s kubectl \
  -n argocd \
  get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' \
  | base64 -d

echo
```

Đăng nhập:

```text
Username: admin
Password: mật khẩu vừa lấy
```

Đổi mật khẩu ngay trong giao diện Argo CD:

```text
User Info → Update Password
```

Sau khi xác nhận đăng nhập thành công bằng mật khẩu mới, xóa initial secret:

```bash
sudo k3s kubectl \
  -n argocd \
  delete secret argocd-initial-admin-secret
```

---

## Phần C — Cài Argo CD production trên Server 3

Thực hiện lại bước 10–13 trên Server 3.

Khi tạo SSH tunnel production, dùng port khác để tránh nhầm.

Trên Server 3:

```bash
sudo k3s kubectl \
  -n argocd \
  port-forward \
  svc/argocd-server \
  8444:443 \
  --address 127.0.0.1
```

Trên máy cá nhân:

```bash
ssh -L 8444:127.0.0.1:8444 ubuntu@IP_SERVER_3
```

Truy cập:

```text
https://localhost:8444
```

Dùng mật khẩu quản trị khác với staging.

## Trạng thái cần đạt sau bước này

- Jenkins hoạt động tại `http://IP_SERVER_1:8080`.
- Jenkins controller có `0 executors`.
- Argo CD staging chỉ quản lý cluster staging.
- Argo CD production chỉ quản lý cluster production.
- Cả hai Argo CD chưa public ra Internet.
- Mật khẩu mặc định đã được thay đổi.
- Jenkins chưa có kubeconfig và chưa thể tác động trực tiếp lên cluster.

## Bước tiếp theo

Cấu hình Jenkins build agent, kết nối GitHub, sau đó cài Harbor và tạo pipeline đầu tiên theo luồng:

```text
GitHub → Jenkins → Harbor → GitOps repository → Argo CD
```
