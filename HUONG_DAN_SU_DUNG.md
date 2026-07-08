# 📘 Hướng Dẫn Sử Dụng GitOps Repository — Argo CD (App-of-Apps)

> Repository: `gitops-repo` — Quản lý Kubernetes cluster bằng Git làm nguồn sự thật duy nhất.

---

## 📑 Mục lục

1. [Giới thiệu](#-giới-thiệu)
2. [Yêu cầu hệ thống](#-yêu-cầu-hệ-thống)
3. [Cài đặt Argo CD lên Cluster](#-cài-đặt-argo-cd-lên-cluster)
4. [Bootstrap Root Application](#-bootstrap-root-application)
5. [Thêm Service Mới](#-thêm-service-mới)
6. [Thêm Infrastructure Mới](#-thêm-infrastructure-mới)
7. [Quản Lý Nhiều Môi Trường](#-quản-lý-nhiều-môi-trường)
8. [Thao Tác Hàng Ngày](#-thao-tác-hàng-ngày)
9. [Xử Lý Sự Cố Thường Gặp](#-xử-lý-sự-cố-thường-gặp)
10. [Kiến Trúc Chi Tiết](#-kiến-trúc-chi-tiết)

---

## 🎯 Giới thiệu

Dự án này sử dụng **Argo CD** với mô hình **App-of-Apps** để:

- Tự động đồng bộ ứng dụng lên Kubernetes cluster
- Git là single source of truth — mọi thay đổi đều qua Pull Request
- Hỗ trợ đa môi trường: `dev`, `staging`, `production`
- Tự động phát hiện và sửa "drift" (cấu hình trên cluster bị thay đổi ngoài ý muốn)

---

## 🔧 Yêu cầu hệ thống

| Công cụ | Phiên bản tối thiểu | Mục đích |
|---------|---------------------|----------|
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | v1.28+ | Tương tác với Kubernetes cluster |
| [argocd CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) | v2.10+ | Quản lý Argo CD |
| [kustomize](https://kustomize.io/) | v5.0+ | (Tích hợp sẵn trong kubectl) |

Kiểm tra:

```bash
kubectl version --client
argocd version --client
```

---

## 🚀 Cài đặt Argo CD lên Cluster

### Bước 1: Tạo namespace và cài Argo CD

```bash
# Tạo namespace
kubectl create namespace argocd

# Cài đặt Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Bước 2: Lấy mật khẩu admin

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> Mặc định username là `admin`.

### Bước 3: Truy cập Argo CD UI

**Cách 1 — Port-forward:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Mở trình duyệt: `https://localhost:8080`

**Cách 2 — LoadBalancer (nếu dùng values của repo):**
```bash
kubectl get svc -n argocd argocd-server
# Lấy External-IP và truy cập qua HTTPS
```

### Bước 4: Đăng nhập bằng CLI (tuỳ chọn)

```bash
argocd login localhost:8080 --insecure
# Nhập username: admin, password: <lấy ở Bước 2>
```

---

## 🌱 Bootstrap Root Application

Sau khi Argo CD đã chạy, bạn chỉ cần một lệnh duy nhất để khởi tạo toàn bộ hệ thống:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

Hoặc nếu bạn có nhiều cluster, dùng Kustomize:

```bash
# Dev cluster
kubectl apply -k clusters/dev/

# Staging cluster
kubectl apply -k clusters/staging/

# Production cluster
kubectl apply -k clusters/production/
```

### Root App sẽ làm gì?

Root App quét thư mục `apps/` (với `recurse: true`) và tự động tạo/triển khai các **child applications**:
- `frontend` → namespace `dev`
- `backend` → namespace `dev`
- `monitoring` → namespace `monitoring`
- `ingress-nginx` → namespace `ingress-nginx`
- `cert-manager` → namespace `cert-manager`

> **Lưu ý:** Trước khi apply, bạn cần đổi `YOUR_ORG` trong các file `application.yaml` thành GitHub/GitLab organization thực tế của bạn.

---

## ➕ Thêm Service Mới

Quy trình thêm một service mới (ví dụ: `payment-service`):

### 1. Tạo cấu trúc thư mục

```bash
# Tạo thư mục cho service mới
mkdir -p apps/payment-service/overlays/{dev,staging,production}
```

### 2. Tạo Application YAML

Tạo file `apps/payment-service/application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_ORG/payment-service-source.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
```

### 3. Đăng ký vào Kustomization

Thêm vào file `apps/kustomization.yaml`:

```yaml
resources:
  - frontend/application.yaml
  - backend/application.yaml
  - payment-service/application.yaml          # <-- thêm dòng này
  - ../infrastructure/monitoring/application.yaml
  - ../infrastructure/ingress-nginx/application.yaml
  - ../infrastructure/cert-manager/application.yaml
```

### 4. Commit và Push

```bash
git add apps/payment-service/
git commit -m "feat: add payment-service application"
git push
```

> Argo CD sẽ tự động phát hiện thay đổi và đồng bộ service mới!

---

## 🏗️ Thêm Infrastructure Mới

Ví dụ: thêm một infrastructure component mới (ví dụ: `redis`).

### 1. Tạo thư mục và Application

```bash
mkdir -p infrastructure/redis/overlays/{dev,staging,production}
```

Tạo `infrastructure/redis/application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_ORG/infra-source.git
    targetRevision: main
    path: redis/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: redis
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 2. Đăng ký vào infrastructure Kustomization

Thêm vào `infrastructure/kustomization.yaml`:

```yaml
resources:
  - monitoring/application.yaml
  - ingress-nginx/application.yaml
  - cert-manager/application.yaml
  - redis/application.yaml                  # <-- thêm dòng này
```

### 3. Đăng ký vào apps Kustomization (để Root App quản lý)

Thêm vào `apps/kustomization.yaml`:

```yaml
  - ../infrastructure/redis/application.yaml
```

---

## 🌐 Quản Lý Nhiều Môi Trường

### Mỗi service có 3 overlay

```
apps/backend/
├── application.yaml          # Application chung, trỏ path đến overlays/dev
└── overlays/
    ├── dev/                  # Môi trường phát triển
    ├── staging/              # Môi trường kiểm thử
    └── production/           # Môi trường sản xuất
```

### Override cho từng môi trường

Mỗi overlay chứa `kustomization.yaml` với các patch riêng:

**Ví dụ `apps/backend/overlays/dev/kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: replica-count.yaml
```

**File patch `replica-count.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1    # Dev chỉ cần 1 replica
```

> **Mẹo:** production nên có replica >= 3, resource limits cao hơn.

### Chuyển môi trường cho một service

Để deploy backend lên staging, sửa `apps/backend/application.yaml`:

```yaml
spec:
  source:
    path: overlays/staging   # Đổi từ dev → staging
```

---

## 📋 Thao Tác Hàng Ngày

### Qua Argo CD UI (Web)

| Thao tác | Cách làm |
|----------|----------|
| Xem trạng thái apps | Dashboard → click vào app |
| Đồng bộ thủ công | Chọn app → **SYNC** |
| Xem lịch sử deploy | Chọn app → **APP DETAILS** → **HISTORY** |
| Rollback | Chọn app → **HISTORY** → chọn revision → **ROLLBACK** |
| Xem resource chi tiết | Click vào từng resource (Pod, Service, v.v.) |

### Qua Argo CD CLI

```bash
# Xem danh sách applications
argocd app list

# Đồng bộ một app
argocd app sync backend

# Xem trạng thái chi tiết
argocd app get backend

# Xem log
argocd app logs backend

# Rollback về revision trước
argocd app rollback backend --rollback-revision 1

# Xoá app (không xoá resource cluster)
argocd app delete backend
```

### Qua Kubernetes CLI

```bash
# Kiểm tra trạng thái Argo CD
kubectl get apps -n argocd

# Xem root app
kubectl get application root -n argocd -o yaml

# Force sync bằng cách xoá app (Argo CD sẽ tạo lại)
kubectl delete application backend -n argocd
```

---

## ⚠️ Xử Lý Sự Cố Thường Gặp

### 1. App bị **OutOfSync**

**Nguyên nhân:** Cluster có resource khác với Git.

**Cách xử lý:**
- Argo CD sẽ tự sync nếu `selfHeal: true`
- Hoặc sync thủ công: `argocd app sync backend`

### 2. Sync **Failed**

**Nguyên nhân:** YAML lỗi, không tìm thấy image, thiếu quyền.

**Cách xử lý:**
```bash
argocd app get backend    # Xem lỗi chi tiết
argocd app logs backend   # Xem log
```
Sửa lỗi trong source repo → commit → push → Argo CD tự sync lại.

### 3. App bị **Missing**

**Nguyên nhân:** Application YAML bị xoá hoặc chưa apply.

**Cách xử lý:**
```bash
kubectl apply -f apps/backend/application.yaml
```

### 4. **CrashLoopBackOff** ở Pod

**Nguyên nhân:** Bug code, thiếu ConfigMap, thiếu Secret.

**Cách xử lý:**
```bash
kubectl logs -n dev <pod-name>
kubectl describe pod -n dev <pod-name>
```

> **Quan trọng:** Argo CD chỉ đảm bảo resource tồn tại, không fix lỗi ứng dụng!

### 5. Không kết nối được Source Repo

**Nguyên nhân:** URL sai, thiếu SSH key / token.

**Cách xử lý:**
```bash
argocd repo list                          # Kiểm tra repo đã đăng ký
argocd repo add https://github.com/...    # Thêm repo
```

### 6. Prune xoá mất resource

Argo CD có thể xoá resource nếu chúng không còn trong Git.

**An toàn hơn:** Tắt `prune: true` khi đang debug, hoặc set `PruneLast: true`.

---

## 📐 Kiến Trúc Chi Tiết

### Sơ đồ App-of-Apps

```
                                                                     
    ┌─────────────────────────────────────────────────────────────┐
    │                    Root Application                          │
    │              (bootstrap/root-app.yaml)                       │
    │        Quản lý danh sách tất cả child apps                   │
    └──────────┬────────────┬──────────────┬──────────────────────┘
               │            │              │
     ┌─────────▼──┐  ┌──────▼──────┐  ┌───▼──────────┐
     │  frontend  │  │   backend   │  │  monitoring  │  ...
     │ (apps/)    │  │ (apps/)     │  │ (infra/)     │
     └────────────┘  └─────────────┘  └──────────────┘
```

### Luồng Deploy

```
Developer
    │
    ├── Push code lên source repo (ví dụ: backend-source.git)
    │       │
    │       ▼
    │   Source repo chứa Kustomize/Helm manifests
    │       │
    │       ▼
    ├── Push thay đổi GitOps repo (gitops-repo)
    │       │
    │       ▼
    │   Argo CD phát hiện thay đổi
    │       │
    │       ▼
    │   Argo CD sync manifests lên cluster
    │       │
    │       ▼
    │   Cluster đạt trạng thái mong muốn ✓
    │
    └── (Nếu có ai đó sửa resource trên cluster trực tiếp)
            │
            ▼
        Argo CD phát hiện "drift"
            │
            ▼
        Tự động rollback về Git state (selfHeal)
```

### Giải thích các tham số SyncPolicy

| Tham số | Ý nghĩa |
|---------|---------|
| `prune: true` | Xoá resource trên cluster nếu không còn trong Git |
| `selfHeal: true` | Tự động sửa nếu resource bị thay đổi ngoài Git |
| `CreateNamespace=true` | Tự tạo namespace nếu chưa có |
| `PruneLast=true` | Chỉ xoá resource cũ sau khi resource mới đã ready |

### File `.gitignore` mẫu

Nếu chưa có, thêm file `.gitignore`:

```gitignore
# OS
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
*.swp
*.swo

# Kubernetes
*.secret.yaml
*.key
```

---

## 💡 Mẹo & Best Practices

1. **Luôn dùng Pull Request** cho mọi thay đổi — không push thẳng vào `main`
2. **Commit message có quy tắc**: `feat:`, `fix:`, `chore:`, `infra:`
3. **Kiểm tra YAML trước khi push**:
   ```bash
   kubectl apply --dry-run=client -f apps/backend/application.yaml
   ```
4. **Tận dụng Kustomize patches** thay vì copy toàn bộ manifest cho mỗi môi trường
5. **Phân quyền bằng AppProject** nếu có nhiều team:
   - Mỗi team một project riêng
   - Giới hạn source repo, cluster resource được phép tạo
6. **Set `PruneLast: true`** để tránh downtime khi thay đổi workload
7. **Dùng `argocd app wait`** trong CI/CD pipeline để chờ deploy hoàn tất
8. **Backup Argo CD config** thường xuyên

---

## 🏁 Quick Start (Tóm tắt nhanh)

```bash
# 1. Cài Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Lấy mật khẩu
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# 3. Đổi YOUR_ORG → GitHub org thật trong các file YAML

# 4. Bootstrap
kubectl apply -f bootstrap/root-app.yaml

# 5. Kiểm tra
kubectl get applications -n argocd
# → root, frontend, backend, monitoring, ingress-nginx, cert-manager
```

---

> 📝 **Tài liệu tham khảo thêm:**
> - [Argo CD Documentation](https://argo-cd.readthedocs.io/)
> - [App-of-Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
> - [Kustomize Documentation](https://kustomize.io/)
