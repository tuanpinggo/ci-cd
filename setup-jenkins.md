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
                         │ giao job qua SSH
                         ▼
                  build-agent-01
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
| Build agent IP | `10.10.0.20` |
| Harbor project | `honglam` |
| Tên ứng dụng/image | `myapp` |
| Source repository | `git@github.com:ORG/myapp.git` |
| GitOps repository | `git@github.com:ORG/myapp-gitops.git` |

Harbor nên sử dụng tên miền có DNS record trỏ về Server 1. Không nên dùng HTTP hoặc cấu hình `insecure-registry` trong production.

---

# Phần A — Cài Harbor trên Server 1

## 3. Kiểm tra tài nguyên và port

Harbor chạy bằng Docker Compose. Riêng Harbor được khuyến nghị có khoảng 4 CPU, 8 GB RAM và 160 GB disk. Server 1 còn chạy Jenkins, Vault, monitoring và logging nên tổng tài nguyên thực tế cần cao hơn đáng kể.

Kiểm tra:

```bash
nproc
free -h
df -h
sudo ss -lntp | grep -E ':80 |:443 '
```

Harbor mặc định sử dụng port `80` và `443`. Jenkins hiện dùng port `8080` nên chưa xung đột.

Nếu sau này Jenkins cũng cần chạy tại `https://jenkins.example.com:443`, cần đặt một reverse proxy chung phía trước hoặc cấp thêm IP cho Server 1. Không thể để Harbor Nginx và một Nginx khác cùng bind port `443` trên một IP.

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

## 5. Chuẩn bị TLS cho Harbor

Production nên dùng certificate từ CA tin cậy hoặc CA nội bộ của tổ chức. Chuẩn bị:

```text
/etc/harbor/tls/harbor.crt
/etc/harbor/tls/harbor.key
```

Tạo thư mục và chép certificate:

```bash
sudo install -d -m 0750 /etc/harbor/tls

sudo install -m 0644 /duong-dan/fullchain.pem \
  /etc/harbor/tls/harbor.crt

sudo install -m 0600 /duong-dan/privkey.pem \
  /etc/harbor/tls/harbor.key
```

Kiểm tra certificate có đúng hostname:

```bash
openssl x509 \
  -in /etc/harbor/tls/harbor.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
```

Nếu dùng CA nội bộ, phải cài CA root lên build agent, Server 2 và Server 3. Không sử dụng `--insecure` trong pipeline.

Tham khảo: [Configure HTTPS access to Harbor](https://goharbor.io/docs/main/install-config/configure-https/)

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
  port: 80

https:
  port: 443
  certificate: /etc/harbor/tls/harbor.crt
  private_key: /etc/harbor/tls/harbor.key

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
curl -fsS https://harbor.example.com/api/v2.0/ping
```

Đăng nhập trình duyệt:

```text
https://harbor.example.com
```

Tham khảo: [Run the Harbor installer](https://goharbor.io/docs/edge/install-config/run-installer-script/)

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

Phương án production khuyến nghị:

```text
Server 1: Jenkins controller + Harbor
VM/Server riêng: build-agent-01
```

Nếu chưa có server thứ tư, có thể tạo một VM riêng trên hạ tầng hiện có. Chỉ trong môi trường thử nghiệm mới đặt agent trực tiếp trên Server 1.

Lý do: user nằm trong group `docker` gần như có quyền root trên agent. Nếu agent cùng host với controller, một Jenkinsfile độc hại có thể ảnh hưởng Jenkins, Harbor và những dịch vụ khác trên Server 1.

Các bước bên dưới chạy trên một Ubuntu VM riêng có hostname `build-agent-01`.

## 13. Tạo user và cài công cụ nền

Trên build agent:

```bash
sudo hostnamectl set-hostname build-agent-01
sudo apt update
sudo apt install -y \
  openjdk-21-jre \
  git \
  make \
  jq \
  curl \
  wget \
  gnupg \
  ca-certificates \
  openssh-server

sudo adduser \
  --disabled-password \
  --gecos '' \
  jenkins-agent

sudo systemctl enable --now ssh
```

Kiểm tra:

```bash
java -version
git --version
```

## 14. Cài Docker trên build agent

Thực hiện lại bước cài Docker Engine ở phần Harbor trên build agent, sau đó:

```bash
sudo usermod -aG docker jenkins-agent
sudo systemctl restart docker
sudo -u jenkins-agent -H docker version
```

Nếu lệnh cuối báo permission denied, đăng xuất/đăng nhập lại hoặc reboot agent rồi thử lại.

> Thành viên group `docker` có quyền kiểm soát Docker daemon và gần tương đương root. Chỉ dùng trên agent chuyên dụng, không cấp cho user trên Jenkins controller.

## 15. Cài Trivy trên build agent

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

## 16. Cài Cosign trên build agent

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

## 17. Cài Kustomize trên build agent

Pipeline dùng Kustomize để thay image digest trong GitOps repository:

```bash
KUSTOMIZE_VERSION='5.8.1'

curl -fsSLo /tmp/kustomize.tar.gz \
  "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv${KUSTOMIZE_VERSION}/kustomize_v${KUSTOMIZE_VERSION}_linux_amd64.tar.gz"

tar -xzf /tmp/kustomize.tar.gz -C /tmp
sudo install -m 0755 /tmp/kustomize /usr/local/bin/kustomize
kustomize version
```

## 18. Cho build agent tin cậy Harbor TLS

Nếu Harbor dùng public CA, thường không cần bước này.

Nếu Harbor dùng CA nội bộ, chép CA root lên agent rồi chạy:

```bash
sudo install -m 0644 harbor-ca.crt \
  /usr/local/share/ca-certificates/harbor-ca.crt

sudo update-ca-certificates

sudo install -d -m 0755 \
  /etc/docker/certs.d/harbor.example.com

sudo install -m 0644 harbor-ca.crt \
  /etc/docker/certs.d/harbor.example.com/ca.crt

sudo systemctl restart docker
curl -fsS https://harbor.example.com/api/v2.0/ping
```

## 19. Tạo SSH key cho controller kết nối agent

Trên Server 1:

```bash
sudo -u jenkins install -d \
  -m 0700 \
  /var/lib/jenkins/.ssh

sudo -u jenkins ssh-keygen \
  -t ed25519 \
  -f /var/lib/jenkins/.ssh/build-agent-01 \
  -C 'jenkins-controller-to-build-agent-01' \
  -N ''

sudo cat /var/lib/jenkins/.ssh/build-agent-01.pub
```

Chép đúng public key vừa in sang build agent:

```bash
sudo install -d \
  -m 0700 \
  -o jenkins-agent \
  -g jenkins-agent \
  /home/jenkins-agent/.ssh

echo 'DAN_PUBLIC_KEY_VAO_DAY' \
  | sudo tee /home/jenkins-agent/.ssh/authorized_keys >/dev/null

sudo chown jenkins-agent:jenkins-agent \
  /home/jenkins-agent/.ssh/authorized_keys

sudo chmod 0600 \
  /home/jenkins-agent/.ssh/authorized_keys
```

Lấy SSH host public key của agent:

```bash
sudo cat /etc/ssh/ssh_host_ed25519_key.pub
sudo ssh-keygen \
  -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Đưa host key đã xác minh vào `/var/lib/jenkins/.ssh/known_hosts` trên controller. Không chọn chiến lược `Non verifying Verification Strategy` trong Jenkins.

Kiểm tra từ Server 1:

```bash
sudo -u jenkins ssh \
  -i /var/lib/jenkins/.ssh/build-agent-01 \
  jenkins-agent@BUILD_AGENT_IP \
  'java -version && docker version && trivy --version && cosign version && kustomize version'
```

## 20. Khai báo SSH credential của agent trong Jenkins

Vào:

```text
Manage Jenkins
→ Credentials
→ System
→ Global credentials
→ Add Credentials
```

Cấu hình:

```text
Kind: SSH Username with private key
ID: build-agent-01-ssh
Username: jenkins-agent
Private Key: Enter directly
```

Lấy private key để dán từ Server 1:

```bash
sudo cat /var/lib/jenkins/.ssh/build-agent-01
```

Không gửi private key qua email/chat và không commit vào Git.

## 21. Khai báo node trong Jenkins

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
Launch method: Launch agents via SSH
Host: BUILD_AGENT_IP
Credentials: build-agent-01-ssh
Host Key Verification Strategy: Known hosts file Verification Strategy
```

Lưu và kiểm tra log node. Trạng thái mong muốn:

```text
Agent successfully connected and online
```

Tham khảo: [Using Jenkins agents](https://www.jenkins.io/doc/book/using/using-agents/)

---

# Phần C — Tạo Jenkins Credentials

## 22. Danh sách credential

Vào `Manage Jenkins → Credentials`, tạo các credential sau:

| Credential ID | Loại | Quyền |
|---|---|---|
| `github-source-read` | SSH private key | Chỉ đọc source repository |
| `github-gitops-write` | SSH private key | Chỉ ghi GitOps repository |
| `harbor-robot-ci` | Username with password | Project robot, chỉ pull/push project `honglam` |
| `cosign-private-key` | Secret file | Cosign private key |
| `cosign-public-key` | Secret file | Cosign public key |
| `cosign-key-password` | Secret text | Mật khẩu mã hóa Cosign private key |

Không dùng cùng một GitHub key cho source repository và GitOps repository. Với GitHub, ưu tiên GitHub App; nếu dùng deploy key, source key chỉ read và GitOps key mới có write.

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

## 24. Cấu hình GitHub known host trên agent

Đưa GitHub Ed25519 host key chính thức vào agent:

```bash
sudo -u jenkins-agent install -d \
  -m 0700 \
  /home/jenkins-agent/.ssh

echo 'github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl' \
  | sudo -u jenkins-agent tee -a \
  /home/jenkins-agent/.ssh/known_hosts >/dev/null

sudo chmod 0600 \
  /home/jenkins-agent/.ssh/known_hosts
```

Đối chiếu fingerprint hiện tại trong tài liệu GitHub trước khi thực hiện: [GitHub SSH fingerprints](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints).

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
        SOURCE_REPO   = 'git@github.com:ORG/myapp.git'
        GITOPS_REPO   = 'git@github.com:ORG/myapp-gitops.git'
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

                    sshagent(credentials: ['github-gitops-write']) {
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
Repository URL: git@github.com:ORG/myapp.git
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
- Đặt CPU, RAM và disk quota cho agent VM.
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
