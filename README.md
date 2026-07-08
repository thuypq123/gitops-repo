# GitOps Repository — Argo CD (App-of-Apps)

## Cấu trúc thư mục

```
gitops-repo/
├── bootstrap/                 # Bootstrap cluster với Argo CD
│   ├── root-app.yaml          # Root Application (App-of-Apps)
│   └── argo-cd/               # Helm values cài đặt Argo CD
├── projects/                  # Argo CD AppProjects (RBAC)
├── apps/                      # Định nghĩa Application cho từng service
│   ├── kustomization.yaml     # Gom tất cả Application
│   ├── frontend/              # Service Frontend
│   └── backend/               # Service Backend
├── infrastructure/             # Application cho cơ sở hạ tầng
│   ├── monitoring/            # Prometheus, Grafana
│   ├── ingress-nginx/         # Ingress Controller
│   └── cert-manager/          # Chứng chỉ TLS
└── clusters/                  # Bootstrap config cho từng cluster
    ├── dev/
    ├── staging/
    └── production/
```

## Quy trình sử dụng

### 1. Cài đặt Argo CD lên cluster

```bash
# Tạo namespace
kubectl create namespace argocd

# Cài đặt Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Lấy mật khẩu admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward UI (nếu cần)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2. Bootstrap Root Application

```bash
kubectl apply -f bootstrap/root-app.yaml
```

Root App sẽ tự động đồng bộ tất cả Application được định nghĩa trong `apps/`.

### 3. Thêm service mới

```bash
# Tạo thư mục
mkdir -p apps/my-service/overlays/{dev,staging,production}

# Tạo Application YAML
cat > apps/my-service/application.yaml << EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-org/my-service-source.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
EOF

# Thêm vào kustomization.yaml
echo "  - my-service/application.yaml" >> apps/kustomization.yaml

# Commit và push
git add . && git commit -m "Add my-service" && git push
```

## Giải thích mô hình App-of-Apps

```
Root App
 ├── App: frontend      → sync từ repo frontend-source.git
 ├── App: backend       → sync từ repo backend-source.git
 ├── App: monitoring     → sync từ repo infra-source.git
 ├── App: ingress-nginx  → sync từ repo infra-source.git
 └── App: cert-manager   → sync từ repo infra-source.git
```

- **Root App** — Quản lý danh sách Application (App of Apps)
- **Child App** — Mỗi app đồng bộ Kubernetes manifests từ source code repo riêng

## Luồng hoạt động

1. Developer push code lên source repo
2. Source repo chứa `kustomize.yaml` / `helm chart` → Kustomize/Helm build ra manifests
3. Argo CD phát hiện thay đổi và tự động sync
4. Cluster đạt trạng thái mong muốn (Git là nguồn sự thật duy nhất)

## Lưu ý

- Thay `YOUR_ORG` trong các file YAML bằng tên GitHub/GitLab organization thực tế
- Thay URL repo source code trong `application.yaml` bằng repo thực tế của bạn
- Dùng `selfHeal: true` để Argo CD tự sửa drift
- Dùng `PruneLast: true` để đảm bảo resource cũ được xóa sau khi resource mới đã sẵn sàng
