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
| Production | `fresh.honglam.net` | `api-fresh.honglam.net` | DNS public → public IP của production → Traefik |

### Staging qua Cloudflare Tunnel

Cloudflare public hostname phải chuyển tiếp cả hai domain tới Traefik của staging cluster. Nếu `cloudflared` chạy trong K3s, origin service có thể dùng HTTP nội bộ:

```yaml
ingress:
  - hostname: fresh-stg.honglam.net
    service: http://traefik.kube-system.svc.cluster.local:80
  - hostname: api-fresh-stg.honglam.net
    service: http://traefik.kube-system.svc.cluster.local:80
  - service: http_status:404
```

TLS public được kết thúc tại Cloudflare edge; kết nối từ Tunnel tới Traefik nằm trong mạng nội bộ cluster. Nếu `cloudflared` chạy ngoài K3s, thay origin service bằng địa chỉ HTTP mà máy chạy Tunnel có thể truy cập tới Traefik.

### Production qua public IP

Tạo bản ghi DNS `A`/`AAAA` cho `fresh.honglam.net` và `api-fresh.honglam.net` trỏ tới public IP của production. Port `80` và `443` phải được chuyển tiếp tới Traefik.

Trước khi public production bằng HTTPS, cấp certificate hợp lệ cho cả hai hostname bằng cert-manager/Let's Encrypt hoặc TLS secret được quản lý ngoài Git. Không đưa private key của certificate vào repository.

## Ghi chú runtime

- React thường nhận API URL tại build time. Hãy cấu hình URL backend trong pipeline/Docker build của `hl_fresh`, hoặc bổ sung cơ chế runtime config phù hợp với image thực tế.
- Probe đang dùng TCP để không giả định ứng dụng đã có endpoint health. Khi backend có `/health`, nên đổi sang `httpGet` để readiness phản ánh cả dependency quan trọng.
- Ingress dùng `ingressClassName: traefik`, phù hợp K3s mặc định. Đổi giá trị này nếu cluster dùng ingress controller khác.
- Staging nhận HTTPS tại Cloudflare edge. Production cần bổ sung `spec.tls` sau khi đã xác định cert-manager issuer hoặc tên TLS secret thực tế.
