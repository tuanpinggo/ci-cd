# Fresh Platform GitOps demo

Thư mục này là một GitOps repository mẫu cho hai ứng dụng:

| Ứng dụng | Source repository | Stack | Logical image name |
|---|---|---|---|
| Frontend | `https://github.com/honglam-cds-v3/hl_fresh.git` | ReactJS | `hl-fresh` |
| Backend | `https://github.com/honglam-cds-v3/fresh-portal.git` | NestJS | `fresh-portal` |

Demo tuân theo mục **26. Cấu trúc GitOps repository** trong `setup-jenkins.md`:

- tài nguyên dùng chung nằm trong `base`;
- cấu hình từng môi trường nằm trong `overlays/staging` và `overlays/production`;
- Deployment dùng logical image name;
- overlay ánh xạ image sang Harbor và lưu image bằng digest;
- Jenkins chỉ cập nhật GitOps repository, Argo CD mới là thành phần sync xuống K3s.

## Cấu trúc

```text
gitops/
├── apps/
│   ├── hl-fresh/
│   │   ├── base/
│   │   └── overlays/{staging,production}/
│   └── fresh-portal/
│       ├── base/
│       └── overlays/{staging,production}/
└── argocd/
    ├── staging/
    └── production/
```

Hai ứng dụng dùng chung namespace theo môi trường:

- staging: `fresh-staging`;
- production: `fresh-production`.

## Giá trị phải thay trước khi dùng

1. Thay `harbor.example.com/honglam` trong bốn file overlay bằng Harbor host/project thật.
2. Thay `https://github.com/honglam-cds-v3/fresh-gitops.git` trong các Argo CD Application nếu GitOps repository có URL khác.
3. Xác nhận Docker image frontend phục vụ HTTP ở port `80`, backend NestJS ở port `3000`. Nếu khác, sửa đồng thời Deployment, Service và probe.
4. Chạy pipeline ít nhất một lần để thay digest toàn số `0` bằng digest có thật trước khi bật Argo CD Application.

Digest toàn số `0` chỉ là placeholder đúng định dạng. Nó không trỏ tới image tồn tại.

## Kiểm tra Kustomize

Chạy từ thư mục `gitops`:

```bash
kustomize build apps/hl-fresh/overlays/staging >/dev/null
kustomize build apps/hl-fresh/overlays/production >/dev/null
kustomize build apps/fresh-portal/overlays/staging >/dev/null
kustomize build apps/fresh-portal/overlays/production >/dev/null
```

Cũng có thể dùng `kubectl kustomize <path>` nếu máy chưa cài binary `kustomize` riêng.

## Cấu hình Jenkins cho từng source repository

Giữ stage `Update GitOps repository` như mẫu trong `setup-jenkins.md` và thay các biến sau.

Frontend:

```groovy
SOURCE_REPO = 'https://github.com/honglam-cds-v3/hl_fresh.git'
GITOPS_REPO = 'https://github.com/honglam-cds-v3/fresh-gitops.git'
IMAGE_NAME  = 'hl-fresh'
```

Backend:

```groovy
SOURCE_REPO = 'https://github.com/honglam-cds-v3/fresh-portal.git'
GITOPS_REPO = 'https://github.com/honglam-cds-v3/fresh-gitops.git'
IMAGE_NAME  = 'fresh-portal'
```

Với các giá trị này, lệnh Jenkins sau sẽ tìm đúng overlay:

```bash
OVERLAY_DIR="apps/${IMAGE_NAME}/overlays/${DEPLOY_ENV}"
kustomize edit set image \
  "${IMAGE_NAME}=${IMAGE_REPOSITORY}@${IMAGE_DIGEST}"
```

## Secret không lưu trong Git

Tạo namespace và Harbor pull secret trên từng cluster trước khi sync lần đầu:

```bash
kubectl create namespace fresh-staging

kubectl -n fresh-staging create secret docker-registry harbor-pull \
  --docker-server=harbor.example.com \
  --docker-username='ROBOT_USERNAME' \
  --docker-password='ROBOT_TOKEN'
```

Thực hiện tương tự với namespace `fresh-production` trên production cluster. Nên dùng robot account chỉ có quyền pull.

Backend có thể đọc secret tên `fresh-portal-secret`; reference này được đặt `optional: true` để manifest vẫn hợp lệ khi demo. Tạo secret từ file env ở ngoài repository:

```bash
kubectl -n fresh-staging create secret generic fresh-portal-secret \
  --from-env-file=/secure/path/fresh-portal.staging.env
```

Không commit file env, mật khẩu database, JWT secret hoặc registry token vào repository này. Trong môi trường thật, nên dùng External Secrets/Vault hoặc một cơ chế secret management tương đương.

## Bootstrap Argo CD

Sau khi đã push GitOps repository và pipeline đã ghi digest thật:

```bash
# Trên cluster staging
kubectl apply -k argocd/staging

# Trên cluster production
kubectl apply -k argocd/production
```

Mỗi Argo CD instance trỏ tới `https://kubernetes.default.svc`, tức cluster nơi chính Argo CD đang chạy. Staging và production vì vậy được bootstrap riêng, đúng với kiến trúc trong `setup-ci-cd.md`.

Nếu GitOps repository là private, cấu hình repository credential trong Argo CD trước khi tạo Application. Các manifest Application bật auto-sync, prune và self-heal; production vẫn chỉ thay đổi sau khi Jenkins approval cập nhật overlay production trong Git.

## Ingress và DNS

Các hostname đã được cấu hình như sau:

| Môi trường | Frontend | Backend | Đường truy cập |
|---|---|---|---|
| Staging | `fresh-stg.honglam.net` | `api-fresh-stg.honglam.net` | Cloudflare Tunnel → Traefik |
| Production | `fresh.honglam.net` | `api-fresh.honglam.net` | Cloudflare proxy → public IP của production → Traefik HTTPS |

### Staging qua Cloudflare Tunnel

Nếu `cloudflared` chạy trực tiếp bằng systemd trên node K3s và Traefik đang nghe port `80` của node, cấu hình hai Published application trên Cloudflare là:

| Public hostname | Service URL |
|---|---|
| `fresh-stg.honglam.net` | `http://localhost:80` |
| `api-fresh-stg.honglam.net` | `http://localhost:80` |

Có thể kiểm tra Traefik trước khi tạo route:

```bash
curl -I -H 'Host: fresh-stg.honglam.net' http://127.0.0.1:80
curl -I -H 'Host: api-fresh-stg.honglam.net' http://127.0.0.1:80
```

Hai hostname cùng vào port `80`; Traefik chọn Service frontend/backend dựa trên HTTP Host header.

Nếu dùng tunnel quản lý bằng file `config.yml`:

```yaml
ingress:
  - hostname: fresh-stg.honglam.net
    service: http://localhost:80
  - hostname: api-fresh-stg.honglam.net
    service: http://localhost:80
  - service: http_status:404
```

TLS public được kết thúc tại Cloudflare edge; Tunnel từ Cloudflare tới `cloudflared` vẫn được mã hóa, còn kết nối local tới Traefik dùng HTTP.

`localhost` chỉ đúng khi `cloudflared` chạy trên node hoặc dùng host network. Nếu `cloudflared` chạy thành Pod trong K3s, thay cả hai Service URL bằng:

```text
http://traefik.kube-system.svc.cluster.local:80
```

### Production qua Cloudflare proxy và public IP

1. Tạo bản ghi DNS `A`/`AAAA` cho `fresh.honglam.net` và `api-fresh.honglam.net` trỏ tới public IP của production.
2. Bật Cloudflare Proxy, trạng thái orange cloud, cho cả hai record.
3. Chuyển tiếp port `443` của public IP tới Traefik production.
4. Trong Cloudflare, vào `SSL/TLS → Origin Server → Create Certificate` và tạo Origin CA certificate cho `*.honglam.net`.
5. Lưu certificate và private key ở máy quản trị an toàn dưới tên ví dụ `origin.crt` và `origin.key`. Không commit hai file này.
6. Tạo TLS Secret trực tiếp trên production cluster:

```bash
kubectl create namespace fresh-production --dry-run=client -o yaml +  | kubectl apply -f -

kubectl -n fresh-production create secret tls cloudflare-origin-tls +  --cert=origin.crt +  --key=origin.key
```

Hai production Ingress cùng tham chiếu Secret `cloudflare-origin-tls`. Secret phải nằm trong namespace `fresh-production`; Argo CD không tạo hoặc quản lý private key này.

7. Trong Cloudflare đặt `SSL/TLS encryption mode` thành `Full (strict)` và bật `Always Use HTTPS`.

Cloudflare Origin CA certificate chỉ dành cho hostname luôn bật proxy. Nếu chuyển record sang DNS only, trình duyệt sẽ không tin certificate này; khi đó phải thay bằng certificate công khai như Let's Encrypt.

## Ghi chú runtime

- React thường nhận API URL tại build time. Hãy cấu hình URL backend trong pipeline/Docker build của `hl_fresh`, hoặc bổ sung cơ chế runtime config phù hợp với image thực tế.
- Probe đang dùng TCP để không giả định ứng dụng đã có endpoint health. Khi backend có `/health`, nên đổi sang `httpGet` để readiness phản ánh cả dependency quan trọng.
- Ingress dùng `ingressClassName: traefik`, phù hợp K3s mặc định. Đổi giá trị này nếu cluster dùng ingress controller khác.
- Staging nhận HTTPS tại Cloudflare edge và đi HTTP nội bộ từ `cloudflared` tới Traefik.
- Production Ingress kết thúc TLS tại Traefik bằng Secret `cloudflare-origin-tls` và yêu cầu Cloudflare `Full (strict)`.
