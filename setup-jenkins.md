# Thiết lập Harbor và Jenkins CI pipeline

> Cập nhật: 2026-09-18  
> Phạm vi: cài Harbor trên Server 1, tạo Jenkins build agent và xây dựng pipeline từ GitHub đến GitOps repository.

## 1. Kiến trúc sau khi hoàn thành

```text
Người vận hành chọn branch staging/main trong Jenkins
                         │
                         ▼
                Jenkins Controller
                    (Server 1)
                         │ Jenkins Remoting/WebSocket nội bộ
                         ▼
            build-agent-01 (Server 1)
                         │
        ┌────────────────┼───────────────────┐
        │                │                   │
   Checkout Git     Test/Lint/Scan      Build image
                                             │
                                             ▼
                                 Scan local container image
                                             │
                                             ▼
                                      Push lên Harbor
                                             │
                                             ▼
                                 Lấy digest và ký bằng Cosign
                                             │
                                             ▼
                                  Cập nhật GitOps repository
                                             │
                                             ▼
                                 Argo CD đồng bộ K3s cluster
```

Quy ước branch:

| Source branch | Môi trường | GitOps overlay | Cơ chế |
|---|---|---|---|
| `staging` | Staging | `overlays/staging` | Tự động cập nhật GitOps |
| `main` | Production | `overlays/production` | Yêu cầu phê duyệt trong Jenkins |

Pipeline không đưa kubeconfig vào Jenkins và không chạy `kubectl apply`. Jenkins chỉ cập nhật GitOps repository; Argo CD chịu trách nhiệm triển khai.

## 2. Các giá trị cần chuẩn bị

Thay toàn bộ giá trị ví dụ dưới đây bằng giá trị thật:

| Giá trị | Ví dụ |
|---|---|
| Harbor FQDN | `harbor.example.com` |
| Server 1 IP | `10.10.0.10` |
| Harbor project | `honglam` |
| Tên ứng dụng/image | `myapp` |
| Source repository | `https://github.com/ORG/myapp.git` |
| GitOps repository | `https://github.com/ORG/myapp-gitops.git` |

Client phải truy cập Harbor qua tên miền HTTPS public. Riêng kết nối từ `cloudflared` tới Harbor trên cùng Server 1 sẽ dùng HTTP qua loopback; không cấu hình `insecure-registry` trên Jenkins agent hoặc các node K3s.

---

# Phần A — Cài Harbor trên Server 1

## 3. Kiểm tra tài nguyên và port

Harbor chạy bằng Docker Compose. Riêng Harbor được khuyến nghị có khoảng 4 CPU, 8 GB RAM và 160 GB disk. Server 1 còn chạy Jenkins, Vault, monitoring và logging nên tổng tài nguyên thực tế cần cao hơn đáng kể.

Kiểm tra:

```bash
nproc
free -h
df -h
sudo ss -lntp | grep -E ':80 |:443 |:8080 |:8081 '
```

Trong hướng dẫn này sử dụng sơ đồ port sau:

| Dịch vụ | Port trên Server 1 | Cách truy cập |
|---|---:|---|
| Jenkins | `8080` | Nội bộ hoặc qua hostname riêng |
| Harbor HTTP origin | `8081` | Cloudflare Tunnel kết nối qua loopback |
| Harbor public | `443` tại Cloudflare edge | `https://harbor.example.com` |

`cloudflared` tạo kết nối outbound tới Cloudflare nên bản thân tiến trình này thường không bind port `80` hoặc `443` trên Server 1. Nếu hai port đó đang bận, lệnh `ss` ở trên sẽ cho biết tiến trình thực tế đang sử dụng chúng. Harbor dùng riêng port origin `8081`.

Không mở public inbound `8081` trong cloud firewall/security group. Khi `cloudflared` chạy ngay trên Server 1, Tunnel truy cập origin qua loopback. Docker có thể publish port ra mọi interface, vì vậy vẫn cần chặn `8081` bằng firewall hạ tầng hoặc chain `DOCKER-USER`.

Tham khảo: [Harbor installation prerequisites](https://goharbor.io/docs/edge/install-config/installation-prereqs/)

## 4. Cài Docker Engine và Docker Compose

Trên Server 1:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Thêm Docker repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Kiểm tra:

```bash
sudo systemctl enable --now docker
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

Tham khảo: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

> Docker có thể bypass một số rule UFW khi publish container port. Cần kiểm soát traffic bằng firewall hạ tầng và chain `DOCKER-USER`, không chỉ dựa vào UFW.

## 5. Mô hình TLS với Cloudflare Tunnel

Trong mô hình này, Harbor và `cloudflared` cùng nằm trên Server 1:

```text
Client --HTTPS--> Cloudflare --Tunnel--> cloudflared --HTTP localhost:8081--> Harbor
```

Harbor origin không cần certificate riêng. HTTPS public được kết thúc tại Cloudflare, còn hop từ `cloudflared` tới Harbor chỉ đi qua loopback của Server 1.

Điều kiện an toàn:

- Không mở port `8081` ra Internet.
- Jenkins agent và node K3s dùng `https://harbor.example.com`, không dùng `http://SERVER_1_IP:8081`.
- Không thêm Harbor vào `insecure-registries` của Docker hoặc K3s.
- Nếu sau này chuyển `cloudflared` sang máy khác, phải dùng HTTPS hoặc mạng private được bảo vệ cho kết nối tới origin.

Tham khảo: [Cloudflare Tunnel configuration](https://developers.cloudflare.com/tunnel/configuration/)

## 6. Tải Harbor installer

Tại thời điểm viết tài liệu, bản GA mới nhất là Harbor `v2.15.2`. Không dùng bản `RC` cho production.

```bash
cd /tmp

HARBOR_VERSION='2.15.2'

curl -fLO \
  "https://github.com/goharbor/harbor/releases/download/v${HARBOR_VERSION}/harbor-online-installer-v${HARBOR_VERSION}.tgz"

sudo tar -xzf \
  "harbor-online-installer-v${HARBOR_VERSION}.tgz" \
  -C /opt
```

Harbor được giải nén tại `/opt/harbor`. Giữ lại thư mục này để nâng cấp và quản lý lifecycle.

Tham khảo: [Harbor releases](https://github.com/goharbor/harbor/releases), [Download Harbor installer](https://goharbor.io/docs/main/install-config/download-installer/)

## 7. Cấu hình `harbor.yml`

```bash
cd /opt/harbor
sudo cp harbor.yml.tmpl harbor.yml
sudo chmod 0600 harbor.yml
sudo nano harbor.yml
```

Điều chỉnh các trường chính:

```yaml
hostname: harbor.example.com

http:
  port: 8081

# URL public mà Docker, Jenkins và trình duyệt sử dụng qua Cloudflare Tunnel.
# Giá trị vẫn là HTTPS dù origin phía trên dùng HTTP.
external_url: https://harbor.example.com

# Dùng mật khẩu dài, ngẫu nhiên và không commit file này lên Git.
harbor_admin_password: CHANGE_TO_A_LONG_RANDOM_PASSWORD

database:
  password: CHANGE_TO_ANOTHER_LONG_RANDOM_PASSWORD

data_volume: /data/harbor

trivy:
  ignore_unfixed: false
  skip_update: false
  offline_scan: false
```

Giữ `external_url` là `https://harbor.example.com`: đây là URL mà Docker, Jenkins agent và K3s nhìn thấy ở Cloudflare edge. Giá trị này không phải giao thức kết nối local giữa `cloudflared` và Harbor.

Tạo data directory trên ổ đĩa có đủ dung lượng:

```bash
sudo install -d -m 0750 /data/harbor
```

## 8. Cài Harbor kèm Trivy

```bash
cd /opt/harbor
sudo ./install.sh --with-trivy
```

Kiểm tra:

```bash
cd /opt/harbor
sudo docker compose ps
sudo docker compose logs --tail=100

# Kiểm tra trực tiếp HTTP origin, không đi qua Cloudflare.
curl -fsS http://127.0.0.1:8081/api/v2.0/ping
```

Nếu Harbor đã được cài bằng port cũ, sau khi sửa `harbor.yml` hãy tạo lại cấu hình và container, không xóa data volume:

```bash
cd /opt/harbor
sudo ./prepare --with-trivy
sudo docker compose down
sudo docker compose up -d
sudo docker compose ps
```

### 8.1. Trỏ Cloudflare Tunnel vào Harbor port `8081`

Nếu Tunnel được quản lý trong Cloudflare Dashboard, tạo Public Hostname với các giá trị:

| Trường | Giá trị |
|---|---|
| Subdomain/hostname | `harbor.example.com` |
| Service type | `HTTP` |
| URL | `localhost:8081` |

Nếu dùng Tunnel được quản lý bằng file local, cập nhật `/etc/cloudflared/config.yml`:

```yaml
tunnel: TUNNEL_UUID
credentials-file: /root/.cloudflared/TUNNEL_UUID.json

ingress:
  - hostname: harbor.example.com
    service: http://127.0.0.1:8081

  - service: http_status:404
```

Kiểm tra và restart Tunnel:

```bash
sudo cloudflared tunnel ingress validate
sudo cloudflared tunnel ingress rule https://harbor.example.com
sudo systemctl restart cloudflared
sudo systemctl status cloudflared --no-pager

# Kiểm tra qua Cloudflare Tunnel.
curl -fsS https://harbor.example.com/api/v2.0/ping
```

Trình duyệt truy cập URL public, không thêm port origin:

```text
https://harbor.example.com
```

Docker cũng sử dụng hostname public:

```bash
docker login harbor.example.com
```

> **Lưu ý khi push image qua Cloudflare:** lưu lượng Registry đi qua HTTP proxy của Cloudflare chịu giới hạn kích thước request theo gói (ví dụ Free/Pro là 100 MB và Business là 200 MB tại thời điểm viết tài liệu) và có thể gặp timeout với layer lớn. Nếu Jenkins báo `413` hoặc `524`, nên thiết kế đường truy cập private có TLS cho build agent và các node K3s; không mở HTTP port `8081` ra Internet và không chuyển Docker sang `insecure-registry`.

Tham khảo: [Run the Harbor installer](https://goharbor.io/docs/edge/install-config/run-installer-script/), [Harbor `external_url`](https://goharbor.io/docs/edge/install-config/configure-yml-file/), [Cloudflare Tunnel configuration file](https://developers.cloudflare.com/tunnel/features/locally-managed-tunnels/configuration-file/), [Cloudflare upload limits](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/)

## 9. Tạo project và policy trong Harbor

Đăng nhập bằng `admin`, sau đó:

1. Vào `Projects → New Project`.
2. Tạo project `honglam`.
3. Không chọn `Public`.
4. Vào `Configuration`.
5. Bật tự động scan image sau khi push.
6. Cấu hình ngưỡng chặn vulnerability phù hợp, ban đầu nên chặn `Critical` rồi nâng dần lên `High`.
7. Tạo tag immutability rule cho các tag CI.
8. Tạo retention rule để tránh đầy ổ đĩa.

Chưa bật bắt buộc Cosign ngay. Hãy chạy thành công pipeline ký và verify image ít nhất một lần, sau đó mới bật policy chỉ cho phép pull signed artifact. Nếu bật quá sớm, workload có thể không pull được image trong lúc pipeline ký chưa hoàn chỉnh.

## 10. Tạo Harbor robot account cho Jenkins

Trong project `honglam`:

```text
Robot Accounts → New Robot Account
```

Cấu hình:

```text
Name: jenkins-ci
Expiration: 90 ngày
Permissions:
  - Pull Repository
  - Push Repository
```

Harbor yêu cầu quyền `Push` đi cùng quyền `Pull`. Chỉ tạo project robot account; không dùng tài khoản `admin` hoặc system robot có quyền toàn bộ registry.

Sau khi tạo, Harbor chỉ hiển thị secret một lần. Lưu ngay username và secret vào password manager, sau đó đưa vào Jenkins Credentials.

Tham khảo: [Harbor project robot accounts](https://goharbor.io/docs/main/working-with-projects/project-configuration/create-robot-accounts/)

## 11. Các lệnh quản trị Harbor cơ bản

```bash
cd /opt/harbor

# Xem trạng thái
sudo docker compose ps

# Xem log
sudo docker compose logs -f --tail=200

# Dừng/start
sudo docker compose stop
sudo docker compose start
```

Phải backup tối thiểu:

- `/opt/harbor/harbor.yml`;
- certificate và private key;
- `/data/harbor`;
- Harbor database;
- robot account inventory và quy trình rotate secret.

Không lưu bản backup duy nhất trên Server 1.

---

# Phần B — Tạo Jenkins build agent

## 12. Vị trí đặt agent

Mô hình áp dụng trong tài liệu này:

```text
Server 1
├── Jenkins controller — user jenkins
├── Jenkins local agent — user jenkins-agent
├── Docker Engine
└── Harbor
```

Agent vẫn là một Jenkins node riêng, có executor, label và workspace riêng, nhưng tiến trình agent chạy ngay trên Server 1. Agent chủ động kết nối tới controller qua WebSocket nội bộ `http://127.0.0.1:8080`; không cần SSH hoặc địa chỉ IP riêng cho node.

Đặt số executor của built-in node về `0` để job không chạy bằng user dịch vụ `jenkins`:

```text
Manage Jenkins
→ Nodes
→ Built-In Node
→ Configure
→ Number of executors: 0
```

> **Rủi ro chấp nhận trong mô hình này:** `jenkins-agent` thuộc group `docker`, nên gần như có quyền root trên Server 1. Việc tách user và workspace giúp tách vận hành nhưng không tạo cách ly bảo mật với Jenkins controller, Harbor, Vault hoặc các dịch vụ khác. Production có yêu cầu bảo mật cao vẫn nên chuyển agent sang VM riêng.

## 13. Tạo user và cài công cụ nền

Thực hiện trên Server 1:

```bash
sudo apt update
sudo apt install -y \
  openjdk-21-jre \
  git \
  make \
  jq \
  curl \
  wget \
  gnupg \
  ca-certificates

if ! id jenkins-agent >/dev/null 2>&1; then
  sudo adduser \
    --disabled-password \
    --gecos '' \
    jenkins-agent
fi

sudo install -d \
  -m 0750 \
  -o jenkins-agent \
  -g jenkins-agent \
  /home/jenkins-agent
```

Kiểm tra:

```bash
java -version
git --version
```

## 14. Cấp quyền Docker cho local agent

Docker Engine đã được cài trên Server 1 ở phần Harbor. Không cài thêm Docker daemon và không cần restart Docker. Chỉ thêm local agent vào group `docker`:

```bash
sudo usermod -aG docker jenkins-agent
id jenkins-agent
sudo -iu jenkins-agent docker version
```

Nếu systemd agent đang chạy từ trước, restart `jenkins-agent.service` sau khi thêm group để tiến trình nhận supplementary group mới.

> Không thêm user `jenkins` của controller vào group `docker`. Chỉ user `jenkins-agent` được phép chạy Docker.

## 15. Cài Trivy cho local agent

```bash
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/trivy.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt update
sudo apt install -y trivy
trivy --version
```

Tham khảo: [Install Trivy](https://www.trivy.dev/docs/latest/getting-started/installation/)

## 16. Cài Cosign cho local agent

Pin phiên bản đã kiểm thử thay vì tự động lấy `latest` trong mỗi build:

```bash
COSIGN_VERSION='3.1.3'

curl -fsSLo /tmp/cosign.deb \
  "https://github.com/sigstore/cosign/releases/download/v${COSIGN_VERSION}/cosign_${COSIGN_VERSION}_amd64.deb"

sudo dpkg -i /tmp/cosign.deb
cosign version
```

Nếu agent dùng ARM64, thay asset `amd64` bằng `arm64` phù hợp với release.

Tham khảo: [Install Cosign](https://docs.sigstore.dev/cosign/system_config/installation/)

## 17. Cài Kustomize cho local agent

Pipeline dùng Kustomize để thay image digest trong GitOps repository:

```bash
KUSTOMIZE_VERSION='5.8.1'

curl -fsSLo /tmp/kustomize.tar.gz \
  "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv${KUSTOMIZE_VERSION}/kustomize_v${KUSTOMIZE_VERSION}_linux_amd64.tar.gz"

tar -xzf /tmp/kustomize.tar.gz -C /tmp
sudo install -m 0755 /tmp/kustomize /usr/local/bin/kustomize
kustomize version
```

## 18. Kiểm tra build agent kết nối Harbor

Local agent có thể nhìn thấy Harbor origin tại `127.0.0.1:8081`, nhưng pipeline vẫn phải sử dụng hostname chuẩn `harbor.example.com`. Nhờ vậy image reference được ghi vào GitOps repository cũng là địa chỉ mà Server 2 và Server 3 có thể pull:

```bash
curl -fsS https://harbor.example.com/api/v2.0/ping
docker login harbor.example.com
```

Agent nhận chứng chỉ HTTPS public ở Cloudflare edge nên không cần cài CA của Harbor origin. Giữ biến pipeline:

```text
HARBOR_HOST=harbor.example.com
```

Không dùng `localhost:8081` làm tên image trong pipeline: image mang tên đó sẽ không thể được K3s trên Server 2 hoặc Server 3 pull. HTTP port `8081` chỉ dành cho `cloudflared` và kiểm tra origin cục bộ.

## 19. Khai báo local node trong Jenkins

Vào:

```text
Manage Jenkins
→ Nodes
→ New Node
```

Cấu hình:

```text
Node name: build-agent-01
Type: Permanent Agent
Number of executors: 1
Remote root directory: /home/jenkins-agent
Labels: linux docker trivy cosign kustomize
Usage: Only build jobs with label expressions matching this node
Launch method: Launch agent by connecting it to the controller
```

Lưu node. Jenkins sẽ hiển thị lệnh kết nối có các giá trị:

```text
Jenkins URL: http://127.0.0.1:8080/
Agent name: build-agent-01
Secret: chuỗi bí mật dành riêng cho node
```

Không chạy trực tiếp lệnh chứa secret trong shell vì secret sẽ xuất hiện trong shell history. Bước tiếp theo lưu secret vào file chỉ local agent được đọc.

## 20. Cài Jenkins Remoting và lưu agent secret

Tải đúng phiên bản `agent.jar` do controller hiện tại cung cấp:

```bash
sudo install -d \
  -m 0755 \
  /opt/jenkins-agent

sudo curl -fsSL \
  http://127.0.0.1:8080/jnlpJars/agent.jar \
  -o /opt/jenkins-agent/agent.jar

sudo chown root:root \
  /opt/jenkins-agent/agent.jar

sudo chmod 0644 \
  /opt/jenkins-agent/agent.jar
```

Tạo file secret:

```bash
sudo install -d \
  -m 0750 \
  -o root \
  -g jenkins-agent \
  /etc/jenkins-agent

sudo touch /etc/jenkins-agent/secret
sudo chown root:jenkins-agent \
  /etc/jenkins-agent/secret
sudo chmod 0640 \
  /etc/jenkins-agent/secret

sudo nano /etc/jenkins-agent/secret
```

Dán duy nhất chuỗi `Secret` lấy từ trang node vào file rồi lưu. Không thêm secret vào tài liệu, Git hoặc Jenkinsfile.

Kiểm tra file JAR và các công cụ dưới đúng user agent:

```bash
sudo -iu jenkins-agent java \
  -jar /opt/jenkins-agent/agent.jar \
  -version

sudo -iu jenkins-agent bash -lc \
  'docker version && trivy --version && cosign version && kustomize version'
```

## 21. Chạy local agent bằng systemd

Tạo `/etc/systemd/system/jenkins-agent.service`:

```ini
[Unit]
Description=Jenkins local build agent
After=jenkins.service docker.service network-online.target
Wants=jenkins.service docker.service network-online.target

[Service]
Type=simple
User=jenkins-agent
Group=jenkins-agent
SupplementaryGroups=docker
WorkingDirectory=/home/jenkins-agent
ExecStart=/usr/bin/java -jar /opt/jenkins-agent/agent.jar \
  -url http://127.0.0.1:8080/ \
  -secret @/etc/jenkins-agent/secret \
  -name build-agent-01 \
  -webSocket \
  -workDir /home/jenkins-agent
Restart=always
RestartSec=10
UMask=0027

[Install]
WantedBy=multi-user.target
```

Nạp và khởi động service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now jenkins-agent
sudo systemctl status jenkins-agent --no-pager
```

Theo dõi log nếu agent chưa online:

```bash
sudo journalctl \
  -u jenkins-agent \
  -n 100 \
  --no-pager
```

Trở lại:

```text
Manage Jenkins
→ Nodes
→ build-agent-01
```

Trạng thái mong muốn:

```text
Agent successfully connected and online
```

WebSocket dùng cùng HTTP port `8080` trên loopback nên không cần mở inbound agent TCP port và không cần SSH credential cho node. Khi nâng cấp Jenkins, tải lại `agent.jar` từ controller rồi restart service:

```bash
sudo curl -fsSL \
  http://127.0.0.1:8080/jnlpJars/agent.jar \
  -o /opt/jenkins-agent/agent.jar

sudo systemctl restart jenkins-agent
```

Tham khảo: [Using Jenkins agents](https://www.jenkins.io/doc/book/using/using-agents/), [Jenkins Remoting inbound agent](https://github.com/jenkinsci/remoting/blob/master/docs/inbound-agent.md)

---

# Phần C — Tạo Jenkins Credentials

## 22. Danh sách credential

Pipeline trong tài liệu dùng GitHub Personal Access Token qua HTTPS, không dùng SSH deploy key. Kiểm tra Jenkins đã có các plugin:

```text
Manage Jenkins
→ Plugins
→ Installed plugins
→ Git
→ Credentials Binding
```

Tất cả credential bên dưới được tạo tại:

```text
Manage Jenkins
→ Credentials
→ System
→ Global credentials (unrestricted)
→ Add Credentials
```

`ID` là giá trị Jenkinsfile tham chiếu và phân biệt chữ hoa/chữ thường. Phải nhập đúng ID trong bảng, không để Jenkins tự sinh ID:

| Credential ID | Loại | Quyền |
|---|---|---|
| `github-source-read` | Username with password | GitHub token chỉ đọc source repository |
| `github-gitops-write` | Username with password | GitHub token đọc/ghi GitOps repository |
| `harbor-robot-ci` | Username with password | Project robot, chỉ pull/push project `honglam` |
| `cosign-private-key` | Secret file | Cosign private key |
| `cosign-public-key` | Secret file | Cosign public key |
| `cosign-key-password` | Secret text | Mật khẩu mã hóa Cosign private key |

### 22.1. Kiểm tra quyền GitHub token

Khuyến nghị dùng hai fine-grained token riêng:

| Token | Repository access | Repository permissions |
|---|---|---|
| Source token | Chỉ chọn source repository | `Contents: Read-only` |
| GitOps token | Chỉ chọn GitOps repository | `Contents: Read and write` |

`Metadata: Read-only` được GitHub cấp kèm theo. Nếu repository thuộc organization, token có thể ở trạng thái `Pending` cho tới khi org owner phê duyệt; organization dùng SSO cũng có thể yêu cầu authorize token.

Nếu hiện tại chỉ có một token, có thể tạm tạo cả hai Jenkins credential bằng cùng token, với điều kiện token truy cập được cả hai repository và có quyền ghi GitOps repository. Cách này hoạt động nhưng quyền rộng hơn cần thiết; nên tách token sau khi pipeline chạy ổn định.

Tài khoản sở hữu token phải có quyền đọc source repository và quyền `Write` trên GitOps repository. Nếu branch `main` của GitOps repository chặn direct push, stage cập nhật GitOps sẽ thất bại; khi đó cần chuyển quy trình sang tạo pull request.

### 22.2. Tạo `github-source-read`

Tại màn hình `Add Credentials`, nhập:

```text
Kind: Username with password
Scope: Global
Username: TEN_DANG_NHAP_GITHUB
Password: Dán GitHub Personal Access Token
ID: github-source-read
Description: GitHub source repository - read only
```

`Username` là GitHub login, không phải email. Token được dán vào ô `Password`; không chọn loại `SSH Username with private key`. Nhấn `Create`.

### 22.3. Tạo `github-gitops-write`

Chọn `Add Credentials` lần nữa và nhập:

```text
Kind: Username with password
Scope: Global
Username: TEN_DANG_NHAP_GITHUB
Password: Dán GitHub Personal Access Token có Contents: Read and write
ID: github-gitops-write
Description: GitHub GitOps repository - read and write
```

Nhấn `Create`. Không đưa token vào URL repository, Jenkinsfile, shell command, description hoặc Git.

### 22.4. Tạo `harbor-robot-ci`

Mở project `honglam` trong Harbor, vào `Robot Accounts` và lấy username/secret của robot account đã tạo ở phần trước. Trong Jenkins chọn `Add Credentials`:

```text
Kind: Username with password
Scope: Global
Username: Username robot chính xác Harbor hiển thị
Password: Secret của robot account
ID: harbor-robot-ci
Description: Harbor project honglam - CI push and pull
```

Không tự sửa prefix username robot. Harbor chỉ hiển thị secret một lần; nếu đã mất secret thì tạo hoặc refresh robot secret rồi cập nhật Jenkins credential.

### 22.5. Tạo ba Cosign credential

Sau khi tạo `cosign.key` và `cosign.pub` theo mục 23, thêm từng credential.

Private key:

```text
Kind: Secret file
Scope: Global
File: Chọn file cosign.key
ID: cosign-private-key
Description: Cosign encrypted private key
```

Public key:

```text
Kind: Secret file
Scope: Global
File: Chọn file cosign.pub
ID: cosign-public-key
Description: Cosign public key
```

Mật khẩu mã hóa private key:

```text
Kind: Secret text
Scope: Global
Secret: Mật khẩu đã nhập khi chạy cosign generate-key-pair
ID: cosign-key-password
Description: Cosign private key password
```

Sau khi hoàn tất, trang `Global credentials` phải hiển thị đủ sáu ID trong bảng. Jenkins che nội dung secret; không kiểm tra bằng cách in credential ra console.

## 23. Tạo Cosign key pair

Chạy trên một máy quản trị tin cậy có Cosign, không chạy trong Jenkins workspace:

```bash
umask 077
cosign generate-key-pair
```

Cosign yêu cầu đặt password và tạo:

```text
cosign.key
cosign.pub
```

Đưa hai file và password vào Jenkins Credentials theo bảng trên. Giữ một bản backup private key được mã hóa ở nơi an toàn. Sau khi Vault hoạt động, nên chuyển signing key sang Vault/KMS thay vì giữ key file dài hạn trong Jenkins. Cosign hỗ trợ URI dạng `hashivault://KEY`.

Tham khảo: [Cosign container signing](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)

## 24. Cấu hình Git tool cho HTTPS credential

Vì GitHub dùng HTTPS + token nên không cần thêm GitHub SSH host key vào `known_hosts`.

Kiểm tra Git trên local agent:

```bash
sudo -iu jenkins-agent git --version
```

Trong Jenkins vào:

```text
Manage Jenkins
→ Tools
→ Git installations
```

Đảm bảo có Git installation:

```text
Name: Default
Path to Git executable: git
```

Nếu Jenkins của bạn đặt tên Git installation khác `Default`, thay giá trị `gitToolName: 'Default'` trong Jenkinsfile bằng đúng tên đó.

Nếu pipeline báo `No such DSL method 'gitUsernamePassword'`, cập nhật plugin `Git` và `Credentials Binding`, sau đó restart Jenkins an toàn.

Tham khảo: [Jenkins Git step](https://www.jenkins.io/doc/pipeline/steps/git/), [Git username/password credentials binding](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/), [GitHub personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

---

# Phần D — Chuẩn bị source và GitOps repository

## 25. Chuẩn hóa lệnh test/lint

Pipeline mẫu gọi ba target:

```bash
make ci-lint
make ci-test
```

Source repository cần có `Makefile` ánh xạ sang công cụ thật của dự án. Ví dụ Node.js:

```makefile
.PHONY: ci-lint ci-test

ci-lint:
	npm ci
	npm run lint

ci-test:
	npm test -- --ci
```

Với Java/Maven có thể đổi thành:

```makefile
.PHONY: ci-lint ci-test

ci-lint:
	./mvnw spotless:check

ci-test:
	./mvnw test
```

Điều chỉnh lệnh phù hợp dự án trước khi chạy pipeline. Không dùng lệnh giả chỉ để pipeline chuyển trạng thái xanh.

## 26. Cấu trúc GitOps repository

Ví dụ:

```text
myapp-gitops/
└── apps/
    └── myapp/
        ├── base/
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── kustomization.yaml
        └── overlays/
            ├── staging/
            │   └── kustomization.yaml
            └── production/
                └── kustomization.yaml
```

Trong `base/deployment.yaml`, dùng logical image name:

```yaml
containers:
  - name: myapp
    image: myapp
```

Ví dụ `overlays/staging/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: myapp-staging

images:
  - name: myapp
    newName: harbor.example.com/honglam/myapp
    digest: sha256:REPLACE_AFTER_FIRST_BUILD
```

Production sử dụng overlay riêng và namespace riêng. Kubernetes image digest là immutable, nên Argo CD luôn triển khai đúng image đã được pipeline ký và kiểm tra.

Tham khảo: [Kustomize images](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/), [Kubernetes image digests](https://kubernetes.io/docs/concepts/containers/images/)

---

# Phần E — Jenkinsfile hoàn chỉnh

## 27. Thứ tự pipeline chính xác

Thứ tự tối ưu là:

```text
Checkout
→ Lint/Test
→ Scan source/dependencies/secrets/misconfiguration
→ Build image local
→ Scan image local
→ Push image lên Harbor
→ Lấy registry digest
→ Ký và verify image theo digest
→ Cập nhật GitOps repository bằng digest
```

Cosign lưu signature dưới dạng OCI artifact/accessory trong registry, vì vậy image phải được push trước khi ký. Không nên cố ký local image rồi mới push; Harbor cần liên kết chữ ký với artifact đã tồn tại trong registry.

Tham khảo: [Sign artifacts with Cosign in Harbor](https://goharbor.io/docs/main/working-with-projects/working-with-images/sign-images/)

## 28. Tạo `Jenkinsfile`

Đặt file sau trong branch `main` của source repository. Thay các giá trị `ORG`, repository, hostname, project và image name.

```groovy
pipeline {
    agent {
        label 'linux && docker && trivy && cosign && kustomize'
    }

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    parameters {
        choice(
            name: 'BRANCH_NAME',
            choices: ['staging', 'main'],
            description: 'staging triển khai staging; main triển khai production sau khi approve'
        )
    }

    environment {
        SOURCE_REPO   = 'https://github.com/ORG/myapp.git'
        GITOPS_REPO   = 'https://github.com/ORG/myapp-gitops.git'
        HARBOR_HOST   = 'harbor.example.com'
        HARBOR_PROJECT = 'honglam'
        IMAGE_NAME    = 'myapp'
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    env.DEPLOY_ENV = params.BRANCH_NAME == 'main' \
                        ? 'production' \
                        : 'staging'
                }
            }
        }

        stage('Checkout latest commit') {
            steps {
                dir('source') {
                    deleteDir()

                    git(
                        branch: params.BRANCH_NAME,
                        credentialsId: 'github-source-read',
                        url: env.SOURCE_REPO
                    )

                    script {
                        env.GIT_COMMIT_FULL = sh(
                            script: 'git rev-parse HEAD',
                            returnStdout: true
                        ).trim()

                        env.GIT_COMMIT_SHORT = env.GIT_COMMIT_FULL.take(12)
                        env.IMAGE_TAG = "${params.BRANCH_NAME}-${env.GIT_COMMIT_SHORT}-${env.BUILD_NUMBER}"
                        env.IMAGE_REPOSITORY = "${env.HARBOR_HOST}/${env.HARBOR_PROJECT}/${env.IMAGE_NAME}"
                        env.IMAGE_REF = "${env.IMAGE_REPOSITORY}:${env.IMAGE_TAG}"

                        currentBuild.displayName = "#${env.BUILD_NUMBER} ${params.BRANCH_NAME} ${env.GIT_COMMIT_SHORT}"
                    }
                }
            }
        }

        stage('Lint') {
            steps {
                dir('source') {
                    sh '''
                        set -euo pipefail
                        make ci-lint
                    '''
                }
            }
        }

        stage('Test') {
            steps {
                dir('source') {
                    sh '''
                        set -euo pipefail
                        make ci-test
                    '''
                }
            }
        }

        stage('Security scan source') {
            steps {
                dir('source') {
                    sh '''
                        set -euo pipefail

                        trivy fs \
                          --scanners vuln,secret,misconfig \
                          --severity HIGH,CRITICAL \
                          --ignore-unfixed \
                          --exit-code 1 \
                          .
                    '''
                }
            }
        }

        stage('Build image') {
            steps {
                dir('source') {
                    sh '''
                        set -euo pipefail

                        DOCKER_BUILDKIT=1 docker build \
                          --pull \
                          --label "org.opencontainers.image.revision=${GIT_COMMIT_FULL}" \
                          --label "org.opencontainers.image.source=${SOURCE_REPO}" \
                          --tag "${IMAGE_REF}" \
                          .
                    '''
                }
            }
        }

        stage('Scan image and generate SBOM') {
            steps {
                dir('source') {
                    sh '''
                        set -euo pipefail

                        trivy image \
                          --scanners vuln,secret \
                          --severity HIGH,CRITICAL \
                          --ignore-unfixed \
                          --exit-code 1 \
                          "${IMAGE_REF}"

                        trivy image \
                          --format cyclonedx \
                          --output ../sbom.cdx.json \
                          "${IMAGE_REF}"
                    '''
                }
            }
        }

        stage('Approve production') {
            when {
                expression {
                    params.BRANCH_NAME == 'main'
                }
            }

            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Deploy ${env.GIT_COMMIT_SHORT} lên production?",
                        ok: 'Approve production'
                    )
                }
            }
        }

        stage('Push, sign and verify image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-robot-ci',
                        usernameVariable: 'HARBOR_USERNAME',
                        passwordVariable: 'HARBOR_PASSWORD'
                    ),
                    file(
                        credentialsId: 'cosign-private-key',
                        variable: 'COSIGN_KEY'
                    ),
                    file(
                        credentialsId: 'cosign-public-key',
                        variable: 'COSIGN_PUBLIC_KEY'
                    ),
                    string(
                        credentialsId: 'cosign-key-password',
                        variable: 'COSIGN_PASSWORD'
                    )
                ]) {
                    dir('source') {
                        sh '''
                            set -euo pipefail

                            cleanup_registry_login() {
                              docker logout "${HARBOR_HOST}" >/dev/null 2>&1 || true
                            }
                            trap cleanup_registry_login EXIT

                            set +x
                            printf '%s' "${HARBOR_PASSWORD}" \
                              | docker login "${HARBOR_HOST}" \
                                  --username "${HARBOR_USERNAME}" \
                                  --password-stdin
                            set -x

                            docker push "${IMAGE_REF}"

                            IMAGE_DIGEST="$(
                              docker buildx imagetools inspect \
                                "${IMAGE_REF}" \
                                --format '{{json .Manifest}}' \
                                | jq -r '.digest'
                            )"

                            case "${IMAGE_DIGEST}" in
                              sha256:*) ;;
                              *)
                                echo "Không lấy được image digest hợp lệ: ${IMAGE_DIGEST}" >&2
                                exit 1
                                ;;
                            esac

                            IMAGE_WITH_DIGEST="${IMAGE_REPOSITORY}@${IMAGE_DIGEST}"

                            printf '%s\n' "${IMAGE_DIGEST}" \
                              > ../image-digest.txt

                            printf '%s\n' "${IMAGE_WITH_DIGEST}" \
                              > ../image-reference.txt

                            export COSIGN_PASSWORD

                            cosign sign \
                              --yes \
                              --tlog-upload=false \
                              --key "${COSIGN_KEY}" \
                              "${IMAGE_WITH_DIGEST}"

                            cosign verify \
                              --key "${COSIGN_PUBLIC_KEY}" \
                              --insecure-ignore-tlog \
                              "${IMAGE_WITH_DIGEST}"
                        '''
                    }

                    script {
                        env.IMAGE_DIGEST = readFile('image-digest.txt').trim()
                        env.IMAGE_WITH_DIGEST = readFile('image-reference.txt').trim()
                    }
                }
            }
        }

        stage('Update GitOps repository') {
            steps {
                dir('gitops') {
                    deleteDir()

                    withCredentials([
                        gitUsernamePassword(
                            credentialsId: 'github-gitops-write',
                            gitToolName: 'Default'
                        )
                    ]) {
                        sh '''
                            set -euo pipefail

                            git clone \
                              --branch main \
                              --single-branch \
                              "${GITOPS_REPO}" \
                              .

                            OVERLAY_DIR="apps/${IMAGE_NAME}/overlays/${DEPLOY_ENV}"

                            test -f "${OVERLAY_DIR}/kustomization.yaml"

                            cd "${OVERLAY_DIR}"

                            kustomize edit set image \
                              "${IMAGE_NAME}=${IMAGE_REPOSITORY}@${IMAGE_DIGEST}"

                            kustomize build . >/dev/null

                            cd "${WORKSPACE}/gitops"

                            git config user.name 'jenkins-ci'
                            git config user.email 'jenkins-ci@example.com'

                            git add \
                              "apps/${IMAGE_NAME}/overlays/${DEPLOY_ENV}/kustomization.yaml"

                            if git diff --cached --quiet; then
                              echo 'GitOps repository đã chứa đúng digest; không cần commit.'
                              exit 0
                            fi

                            git commit \
                              -m "deploy(${IMAGE_NAME}): ${DEPLOY_ENV} ${GIT_COMMIT_SHORT}"

                            git push origin HEAD:main
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Đã cập nhật ${env.DEPLOY_ENV}: ${env.IMAGE_WITH_DIGEST}"
        }

        always {
            archiveArtifacts(
                artifacts: 'sbom.cdx.json,image-reference.txt',
                allowEmptyArchive: true,
                fingerprint: true
            )

            sh '''
                if [ -n "${IMAGE_REF:-}" ]; then
                  docker image rm -f "${IMAGE_REF}" || true
                fi

                docker logout "${HARBOR_HOST}" >/dev/null 2>&1 || true
            '''

            deleteDir()
        }
    }
}
```

## 29. Lưu ý về pipeline mẫu

1. Thay URL GitHub, domain Harbor, project và image name trước khi chạy.
2. `make ci-lint` và `make ci-test` phải hoạt động trên agent.
3. Scan trả exit code `1` khi có `HIGH/CRITICAL`, do đó build sẽ dừng trước khi push.
4. Tag có dạng `branch-commit-build`, ví dụ `staging-a83c21f093bd-42`; không dùng `latest`.
5. GitOps lưu image theo digest, không theo mutable tag.
6. Với production nghiêm ngặt, nên để pipeline tạo branch/PR cho GitOps production thay vì push trực tiếp vào `main`. Ví dụ trên dùng Jenkins approval để giữ quy trình dễ triển khai ban đầu.
7. Sau khi pipeline ký thành công, có thể bật Cosign content trust trong Harbor project.

---

# Phần F — Tạo Jenkins job có màn hình chọn branch

## 30. Tạo Pipeline job

Trong Jenkins:

```text
New Item
→ Nhập tên: myapp-ci
→ Chọn Pipeline
→ OK
```

Trong cấu hình job:

```text
Pipeline
Definition: Pipeline script from SCM
SCM: Git
Repository URL: https://github.com/ORG/myapp.git
Credentials: github-source-read
Branch Specifier: */main
Script Path: Jenkinsfile
Lightweight checkout: enabled
```

Jenkinsfile luôn được lấy từ branch `main` đáng tin cậy, nhưng pipeline sẽ checkout branch mà người dùng chọn trong parameter.

Lưu job rồi chạy `Build Now` lần đầu để Jenkins đọc phần `parameters`. Từ lần sau giao diện hiển thị:

```text
Build with Parameters
└── BRANCH_NAME
    ├── staging
    └── main
```

## 31. Kiểm tra pipeline lần đầu

Chạy `staging` trước và kiểm tra theo thứ tự:

1. Jenkins checkout đúng commit HEAD của `staging`.
2. Lint và test thành công.
3. Trivy không phát hiện lỗi vượt ngưỡng.
4. Harbor xuất hiện image tag mới.
5. Artifact có Cosign signature/accessory trong Harbor.
6. Jenkins lưu `sbom.cdx.json` và `image-reference.txt`.
7. GitOps staging overlay đổi sang digest mới.
8. Argo CD staging phát hiện commit và sync.

Kiểm tra image thủ công trên agent:

```bash
docker login harbor.example.com

cosign verify \
  --key cosign.pub \
  --insecure-ignore-tlog \
  harbor.example.com/honglam/myapp@sha256:IMAGE_DIGEST
```

Sau khi staging hoạt động ổn định mới chạy `main` và thử bước approval production.

---

# Phần G — Hardening sau khi pipeline chạy ổn định

## 32. Việc nên làm ngay

- Bật HTTPS và không sử dụng `insecure-registry`.
- Rotate Harbor robot secret định kỳ.
- Giới hạn agent còn một executor trong giai đoạn đầu.
- Theo dõi CPU, RAM và disk dùng chung trên Server 1; giới hạn tài nguyên của build/container để CI không làm gián đoạn Harbor và Jenkins.
- Lập kế hoạch chuyển local agent sang VM riêng khi workload hoặc yêu cầu bảo mật tăng.
- Bật tag immutability và retention trong Harbor.
- Bật scan-on-push để Harbor kiểm tra lại image sau CI scan.
- Bật Cosign content trust sau khi đã verify end-to-end.
- Bảo vệ branch `main` của source và GitOps repository.
- Yêu cầu review/approval cho production GitOps change.
- Backup Harbor data/database ngoài Server 1.
- Không cấp kubeconfig production cho Jenkins.
- Không mount Docker socket của Server 1 vào Jenkins controller.

## 33. Hướng nâng cấp tiếp theo

Khi Vault được triển khai:

- chuyển Cosign signing key sang Vault Transit/KMS;
- cấp Harbor credential ngắn hạn nếu quy trình hỗ trợ;
- không giữ private signing key lâu dài dưới dạng Jenkins Secret File.

Khi số lượng pipeline tăng:

- thay permanent agent bằng ephemeral VM hoặc Kubernetes agent;
- mỗi build có workspace và môi trường mới;
- tách agent frontend, backend và image-builder bằng label;
- không chạy CI workload trên production K3s cluster.

## 34. Tài liệu tham khảo chính thức

- [Harbor installation prerequisites](https://goharbor.io/docs/edge/install-config/installation-prereqs/)
- [Harbor HTTPS configuration](https://goharbor.io/docs/main/install-config/configure-https/)
- [Harbor installation with Trivy](https://goharbor.io/docs/edge/install-config/run-installer-script/)
- [Harbor robot accounts](https://goharbor.io/docs/main/working-with-projects/project-configuration/create-robot-accounts/)
- [Harbor Cosign integration](https://goharbor.io/docs/main/working-with-projects/working-with-images/sign-images/)
- [Jenkins agents](https://www.jenkins.io/doc/book/using/using-agents/)
- [Trivy installation](https://www.trivy.dev/docs/latest/getting-started/installation/)
- [Cosign installation](https://docs.sigstore.dev/cosign/system_config/installation/)
- [Cosign container signing](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)
- [Docker Engine installation](https://docs.docker.com/engine/install/ubuntu/)
- [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
