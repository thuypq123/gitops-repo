# 📘 Hướng Dẫn Sử Dụng GitOps Repository — Argo CD (App-of-Apps)

> Repository: `gitops-repo` — Quản lý Kubernetes cluster bằng Git làm nguồn sự thật duy nhất.
> Mọi Kubernetes manifest đều nằm trong repo này, **tách biệt hoàn toàn với source code** ứng dụng.

---

## 📑 Mục lục

1. [Giới thiệu](#-giới-thiệu)
2. [Cấu trúc thư mục](#-cấu-trúc-thư-mục)
3. [Yêu cầu hệ thống](#-yêu-cầu-hệ-thống)
4. [Cài đặt Argo CD lên Cluster](#-cài-đặt-argo-cd-lên-cluster)
5. [Truy cập & Host Mapping](#-truy-cập--host-mapping)
6. [Bootstrap Root Application](#-bootstrap-root-application)
7. [Observability Stack (Monitoring & Logging)](#-observability-stack-monitoring--logging)
8. [Thêm Service Mới](#-thêm-service-mới)
9. [Thêm Infrastructure Mới](#-thêm-infrastructure-mới)
10. [Build & Deploy Image Mới](#-build--deploy-image-mới)
11. [Thao Tác Hàng Ngày](#-thao-tác-hàng-ngày)
12. [Xử Lý Sự Cố Thường Gặp](#-xử-lý-sự-cố-thường-gặp)
13. [Kiến Trúc Chi Tiết](#-kiến-trúc-chi-tiết)

---

## 🎯 Giới thiệu

Dự án này sử dụng **Argo CD** với mô hình **App-of-Apps** để:

- Tự động đồng bộ ứng dụng lên Kubernetes cluster
- Git là single source of truth — mọi thay đổi đều qua Pull Request
- Hỗ trợ đa môi trường: `dev`, `staging`, `production`
- Tự động phát hiện và sửa "drift" (cấu hình trên cluster bị thay đổi ngoài ý muốn)

### Luồng GitOps

```
[Developer sửa K8s manifest]
         │
         ▼
   Push lên github.com/thuypq123/gitops-repo.git
         │
         ▼
   Argo CD phát hiện thay đổi (poll 3 phút / webhook)
         │
         ▼
   Sync manifests xuống Minikube cluster
         │
         ▼
   Pod/Service/Ingress được cập nhật ✅
```

### Repository liên quan

| Repo | Mục đích | Nội dung |
|------|----------|----------|
| **thuypq123/gitops-repo** | GitOps (repo này) | K8s manifests, Kustomize, Argo CD config |
| **thuypq123/golang_feature_first** | Source code | Code Go, Dockerfile (không chứa K8s manifest) |

---

## 📂 Cấu trúc thư mục

```
gitops-repo/
├── apps/                                    # ← Root App quản lý danh sách App Argo CD
│   ├── kustomization.yaml                   #   Danh sách child applications
│   ├── backend/                             #   App: backend (payment-service)
│   │   ├── application.yaml                 #     Argo CD Application definition
│   │   ├── base/                            #     Kustomize base
│   │   │   ├── kustomization.yaml
│   │   │   ├── deployment.yaml              #       Deployment + env vars
│   │   │   └── service.yaml                 #       ClusterIP service
│   │   └── overlays/                        #     Môi trường
│   │       ├── dev/kustomization.yaml       #       1 replica
│   │       ├── staging/kustomization.yaml   #       2 replicas
│   │       └── production/kustomization.yaml #      3 replicas
│   └── infrastructure/
│       └── application.yaml                 # App: infrastructure (Redis, Ingress, Observability)
├── infrastructure/                          # Các component hạ tầng
│   ├── kustomization.yaml                   # Danh sách resource
│   ├── redis.yaml                           # Redis Deployment + Service
│   ├── ingress-backend.yaml                 # Ingress cho Backend API (api.local)
│   ├── ingress-grafana.yaml                 # Ingress cho Grafana (grafana.local)
│   ├── ingress-argocd.yaml                  # Ingress cho Argo CD (argocd.local)
│   ├── prometheus.yaml                      # Prometheus + Config + Service + RBAC
│   ├── grafana.yaml                         # Grafana + Datasources (Prometheus/Loki/Tempo)
│   ├── tempo.yaml                           # Tempo (Distributed Tracing)
│   ├── loki.yaml                            # Loki (Log Aggregation) + StatefulSet
│   ├── otel-collector.yaml                  # OpenTelemetry Collector
│   └── fluent-bit.yaml                      # Fluent Bit (Log Shipper → Loki)
├── bootstrap/
│   ├── root-app.yaml                        # Root Application (App-of-Apps)
│   └── argo-cd/values.yaml                  # Helm values cho Argo CD
├── clusters/
│   ├── dev/kustomization.yaml
│   ├── staging/kustomization.yaml
│   └── production/kustomization.yaml
├── projects/
│   └── project.yaml                         # Argo CD AppProject
├── README.md
└── HUONG_DAN_SU_DUNG.md                     # Tài liệu này
```

---

## 🔧 Yêu cầu hệ thống

| Công cụ | Phiên bản tối thiểu | Mục đích |
|---------|---------------------|----------|
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | v1.28+ | Tương tác với Kubernetes cluster |
| [minikube](https://minikube.sigs.k8s.io/) | v1.30+ | Kubernetes cluster local |
| [argocd CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) | v2.10+ | Quản lý Argo CD |
| [kustomize](https://kustomize.io/) | v5.0+ | (Tích hợp sẵn trong kubectl) |
| [docker](https://docs.docker.com/engine/install/) | 24+ | Build images |
| [go](https://go.dev/dl/) | 1.22+ | Build Go app (nếu dev source) |

Kiểm tra:

```bash
kubectl version --client
argocd version --client
minikube version
docker --version
```

---

## 🚀 Cài đặt Argo CD lên Cluster

### Bước 1: Start Minikube

```bash
minikube start --cpus=4 --memory=8192
```

### Bước 2: Enable Ingress Controller

```bash
minikube addons enable ingress
```

### Bước 3: Tạo namespace và cài Argo CD

```bash
# Tạo namespace
kubectl create namespace argocd

# Cài đặt Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Kiểm tra pod đều Running
kubectl get pods -n argocd -w
```

### Bước 4: Lấy mật khẩu admin

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> Mặc định username là `admin`.

### Bước 5: Thêm GitOps repo vào Argo CD

```bash
argocd repo add https://github.com/thuypq123/gitops-repo.git \
  --username thuypq123 --password <your-token>
```

---

## 🔗 Truy cập & Host Mapping

### Thêm host mapping

Sau khi Ingress Controller đã enable, bạn cần thêm host mapping:

```bash
echo "$(minikube ip) api.local argocd.local grafana.local" | sudo tee -a /etc/hosts
```

### Danh sách dịch vụ

| Dịch vụ | URL | Port | Ghi chú |
|---------|-----|------|---------|
| **Argo CD UI** | [`https://argocd.local`](https://argocd.local) | 443 | Bấm Advanced → Proceed (self-signed SSL) |
| **Grafana** | [`http://grafana.local`](http://grafana.local) | 3000 | Dashboard logs/metrics/traces |
| **Backend API** | [`http://api.local/health`](http://api.local/health) | 8080 | Health check |
| **Swagger Docs** | [`http://api.local/swagger/index.html`](http://api.local/swagger/index.html) | 8080 | API documentation |

**Thông tin đăng nhập:**

| Service | Username | Password |
|---------|----------|----------|
| Argo CD | `admin` | (Lấy ở bước 4) |
| Grafana | `admin` | `admin` |

> **Giải thích:** Trên cloud production, thay vì `/etc/hosts` + IP Minikube, bạn sẽ dùng domain thật (VD: `argocd.example.com`) + Load Balancer + cert-manager (Let's Encrypt).

---

## 🌱 Bootstrap Root Application

Sau khi Argo CD đã chạy, bạn chỉ cần một lệnh duy nhất để khởi tạo toàn bộ hệ thống:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

### Root App sẽ làm gì?

Root App quét thư mục `apps/` (với `recurse: true`) và tự động tạo/triển khai các **child applications**:

- `backend` → namespace `dev` — Deploy payment-service
- `infrastructure` → namespace `dev` — Deploy Redis, Ingress, và toàn bộ Observability Stack

> **Lưu ý:** Nếu thêm ứng dụng mới, nhớ thêm vào [apps/kustomization.yaml](file:///home/waltonwalter/Desktop/gitOps/apps/kustomization.yaml).

### Kiểm tra sau bootstrap

```bash
# Qua CLI
argocd app list

# Qua kubectl
kubectl get applications -n argocd

# Kết quả mong đợi:
# NAME             STATUS   HEALTH
# root             Synced   Healthy
# backend          Synced   Healthy
# infrastructure   Synced   Healthy
```

---

## 📊 Observability Stack (Monitoring & Logging)

Stack hiện tại bao gồm **4 thành phần** hoạt động cùng nhau.

### Kiến trúc tổng quan

```
                          ┌──────────────────┐
                          │   Grafana (UI)   │
                          │  grafana:3000    │
                          └──────┬───────────┘
                    ┌────────────┼─────────────┐
                    ▼            ▼             ▼
             ┌──────────┐ ┌──────────┐ ┌──────────┐
             │Prometheus│ │   Loki   │ │  Tempo   │
             │ Metrics  │ │  Logs    │ │ Traces   │
             └─────┬────┘ └────┬─────┘ └────┬─────┘
                   │           │             │
                   │    ┌──────┴──────┐     │
                   │    │ Fluent Bit  │     │
                   │    │ (DaemonSet) │     │
                   │    │ /var/log/*  │     │
                   │    └─────────────┘     │
                   │                        │
                   │    ┌──────────────────┐│
                   └────┤ OTel Collector   ◄┘
                        │ otel-collector:  │
                        │ 4317(gRPC)       │
                        │ 4318(HTTP)       │
                        └──────────────────┘
                                  ▲
                                  │ (OTLP)
                          ┌───────┴───────┐
                          │ Payment-Svc   │
                          │ (Go OTel SDK) │
                          └───────────────┘
```

### 1. Prometheus — Thu thập Metrics

- **File:** [infrastructure/prometheus.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/prometheus.yaml)
- **Image:** `prom/prometheus:v2.53.0`
- **Service:** `prometheus-service:9090`
- **Data:** Chưa có persistent storage (emptyDir)

**Cách hoạt động:**

Prometheus quét (scrape) tất cả pods có annotation `prometheus.io/scrape: "true"` trong cluster.
Kết quả được gán thêm các labels: `kubernetes_namespace`, `kubernetes_pod_name`, và tất cả pod labels.

**Targets hiện tại:**

| Target | Trạng thái | Port | Ghi chú |
|--------|-----------|------|---------|
| Payment service | ✅ **up** | 8080 (pod annotation) | Tự động phát hiện |
| OTel Collector | ✅ **up** | 8888 (static config) | Metrics từ OpenTelemetry |

### 2. Grafana — Dashboard

- **File:** [infrastructure/grafana.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/grafana.yaml)
- **Image:** `grafana/grafana:11.1.0`
- **Service:** `grafana-service:3000`
- **URL:** [`http://grafana.local`](http://grafana.local)

**Datasources đã cấu hình sẵn:**

| Datasource | UID | URL | Mục đích |
|-----------|-----|-----|----------|
| **Prometheus** | `prometheus` | `http://prometheus-service:9090` | Metrics |
| **Loki** | `loki` | `http://loki-service:3100` | Logs |
| **Tempo** | `tempo` | `http://tempo-service:3200` | Traces |

**Cách query trong Grafana (Explore):**

| Mục đích | Query | Datasource |
|----------|-------|-----------|
| Xem payment-service có up không | `up{kubernetes_namespace="dev"}` | Prometheus |
| Số goroutines | `go_goroutines{kubernetes_namespace="dev"}` | Prometheus |
| Memory đang dùng | `go_memstats_alloc_bytes{kubernetes_namespace="dev"}` | Prometheus |
| CPU usage | `rate(process_cpu_seconds_total{kubernetes_namespace="dev"}[1m])` | Prometheus |
| Logs trong namespace dev | `{namespace_name="dev"}` | Loki |
| Logs của payment-service | `{container_name="payment-service"}` | Loki |
| Traces của payment-service | `{service.name="payment-service"}` | Tempo |

### 3. Loki + Fluent Bit — Log Aggregation

- **Fluent Bit:** [infrastructure/fluent-bit.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/fluent-bit.yaml)
- **Loki:** [infrastructure/loki.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/loki.yaml)
- **Image Fluent Bit:** `cr.fluentbit.io/fluent/fluent-bit:3.1.4`
- **Image Loki:** `grafana/loki:3.1.0`

**Cách hoạt động:**

Fluent Bit chạy như DaemonSet, đọc tất cả log files từ `/var/log/containers/*.log`.
Kubernetes filter sử dụng `use_tag_for_meta true` để trích xuất metadata từ đường dẫn file log (namespace, pod, container).
Logs được đẩy lên Loki với các labels: `namespace_name`, `pod_name`, `container_name`, `service_name`.

**Labels có sẵn trong Loki:**

- `job` = `fluentbit`
- `namespace_name` — tên namespace (VD: `dev`, `kube-system`, `argocd`)
- `pod_name` — tên pod (VD: `dev-payment-service-xxx`)
- `container_name` — tên container (VD: `payment-service`)
- `service_name` — tên service từ K8s labels

### 4. Tempo + OTel Collector — Distributed Tracing

- **Tempo:** [infrastructure/tempo.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/tempo.yaml)
- **OTel Collector:** [infrastructure/otel-collector.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/otel-collector.yaml)
- **Image Tempo:** `grafana/tempo:2.6.0`
- **Image OTel:** `otel/opentelemetry-collector-contrib:0.107.0`

**Cách hoạt động:**

Service gửi traces (OTLP protocol) đến **OTel Collector** (port 4317 gRPC / 4318 HTTP).
OTel Collector chuyển tiếp traces đến **Tempo** (port 4317).
Tempo lưu trữ traces (backend local, retention 24h).
Grafana query Tempo để hiển thị traces.

**Service nào có tracing?**
- **Payment-service:** ✅ Đã tích hợp OTel SDK, gửi traces đến `otel-collector:4318`

### Lưu ý

> Dữ liệu hiện tại dùng `emptyDir` (Prometheus, Tempo) và ephemeral storage (Loki PVC 1Gi).
> Nếu restart pod, dữ liệu cũ sẽ mất. Cần gắn PV/PVC cho production.

---

## ➕ Thêm Service Mới

Quy trình thêm một service mới (ví dụ: `order-service`, `notification-service`, v.v.):

### Checklist tổng kết

| # | Bước | Cần làm | Mô tả |
|---|------|---------|-------|
| 1 | Deployment | Tạo file YAML | Phải có annotation & env cho observability |
| 2 | Service | Tạo K8s Service | Để pods giao tiếp nội bộ |
| 3 | **Logs** | ✅ **Tự động** | Fluent Bit tự đọc log |
| 4 | **Metrics** | ✅ **Tự động** | Prometheus tự quét nếu có annotation |
| 5 | **Tracing** | ⚠️ Cần code | Tích hợp OTel SDK + set env |
| 6 | Ingress | Thêm vào ingress YAML | Nếu cần public API |
| 7 | Kustomize | Thêm resource path | Để Argo CD quản lý |

### Bước 1: Tạo cấu trúc thư mục

```bash
# Service mới với Kustomize base + overlays
mkdir -p apps/order-service/base
mkdir -p apps/order-service/overlays/{dev,staging,production}
```

### Bước 2: Tạo Deployment (quan trọng nhất)

Tạo file `apps/order-service/base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        environment: dev
      annotations:
        prometheus.io/scrape: "true"          # ✅ BẮT BUỘC: để Prometheus phát hiện
        prometheus.io/port: "8080"             # Port metrics endpoint
        prometheus.io/path: "/metrics"         # Path metrics endpoint
    spec:
      containers:
        - name: order-service
          image: your-dockerhub/order-service:latest
          ports:
            - containerPort: 8080
          env:
            # ─── Observability ─────────────────────────────
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector:4318"   # Gửi traces đến OTel Collector (HTTP)
              # value: "otel-collector:4317"         # Hoặc gRPC
            - name: OTEL_SERVICE_NAME
              value: "order-service"                 # Tên hiển thị trong Tempo/Grafana
            # ─── Business ─────────────────────────────────
            - name: ENV
              value: "development"
            # ... các env khác
```

> [!IMPORTANT]
> Hai env `OTEL_EXPORTER_OTLP_ENDPOINT` và `OTEL_SERVICE_NAME` là **bắt buộc** để có tracing.
> Annotation `prometheus.io/scrape: "true"` là **bắt buộc** để có metrics.

### Bước 3: Tạo Service

Tạo file `apps/order-service/base/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
```

### Bước 4: Tạo Kustomize base

Tạo file `apps/order-service/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

### Bước 5: Tạo overlay cho từng môi trường

Tạo file `apps/order-service/overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namePrefix: dev-
commonLabels:
  environment: dev
```

### Bước 6: Tạo Application YAML cho Argo CD

Tạo file `apps/order-service/application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/thuypq123/gitops-repo.git
    targetRevision: main
    path: apps/order-service/overlays/dev
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

### Bước 7: Đăng ký vào Root App

Thêm vào [apps/kustomization.yaml](file:///home/waltonwalter/Desktop/gitOps/apps/kustomization.yaml):

```yaml
resources:
  - backend/application.yaml
  - order-service/application.yaml          # <-- thêm dòng này
  - infrastructure/application.yaml
```

### Bước 8: Thêm Ingress (nếu cần public API)

Thêm path vào [infrastructure/ingress-backend.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/ingress-backend.yaml):

```yaml
spec:
  rules:
    - host: api.local
      http:
        paths:
          - path: /api/v1/orders             # Path riêng
            pathType: Prefix
            backend:
              service:
                name: dev-order-service      # K8s Service name
                port:
                  number: 8080
```

### Bước 9: Tích hợp OTel SDK trong code (để có tracing)

Nếu service viết bằng Go:

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/trace"
)

// Khởi tạo tracer khi start app
exporter, _ := otlptracehttp.New(ctx,
    otlptracehttp.WithEndpoint("otel-collector:4318"),
    otlptracehttp.WithInsecure(),
)
tp := trace.NewTracerProvider(trace.WithBatcher(exporter))
otel.SetTracerProvider(tp)
```

> Các ngôn ngữ khác (Java, Python, Node.js, ...) cũng dùng OTLP endpoint tương tự.

### Bước 10: Commit và Push

```bash
git add apps/order-service/
git commit -m "feat: add order-service"
git push
```

> Argo CD sẽ tự động phát hiện thay đổi và đồng bộ service mới!
> Logs → tự động có. Metrics → tự động có. Tracing → có nếu đã tích hợp OTel SDK.

---

## 🏗️ Thêm Infrastructure Mới

Ví dụ: thêm một infrastructure component (VD: `kafka`).

### Cấu trúc hiện tại

Các component infrastructure được quản lý qua [infrastructure/kustomization.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/kustomization.yaml):

```yaml
resources:
  - postgres.yaml
  - redis.yaml
  - ingress-backend.yaml
  - ingress-argocd.yaml
  - ingress-grafana.yaml
  - prometheus.yaml
  - grafana.yaml
  - tempo.yaml
  - loki.yaml
  - otel-collector.yaml
  - fluent-bit.yaml
```

### Thêm component mới

```bash
# Tạo file manifest (Deployment + Service)
touch infrastructure/kafka.yaml
```

Thêm vào [infrastructure/kustomization.yaml](file:///home/waltonwalter/Desktop/gitOps/infrastructure/kustomization.yaml):

```yaml
resources:
  - kafka.yaml               # <-- thêm dòng này
  # ... các resources khác giữ nguyên
```

> **Lưu ý:** Infrastructure không cần tạo Application riêng lẻ — `infrastructure/application.yaml` là một Argo CD App duy nhất quản lý toàn bộ thư mục này.

---

## 🔄 Build & Deploy Image Mới

Khi source code Go thay đổi, cần build image mới:

### Bước 1: Build image từ source code

```bash
# Vào thư mục source
cd /home/waltonwalter/Desktop/golang

# Set Docker vào Minikube
eval $(minikube docker-env)

# Build image
docker build -f deployments/Dockerfile -t thuypq123/payment-service:latest .

# Kiểm tra
docker images | grep payment-service
```

### Bước 2: Restart pod để dùng image mới

```bash
kubectl rollout restart -n dev deploy/dev-payment-service

# Kiểm tra
kubectl rollout status -n dev deploy/dev-payment-service
```

### CI/CD tự động (tương lai)

Khi có GitHub Actions, luồng sẽ tự động:

```
Push code → Build image → Push lên Docker Hub → Argo CD detect → Auto-deploy
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
# Đăng nhập
argocd login argocd.local --insecure
# Nhập username: admin, password: <lấy ở trên>

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
kubectl get applications -n argocd

# Xem root app
kubectl get application root -n argocd -o yaml

# Force sync bằng cách xoá app (Argo CD sẽ tạo lại)
kubectl delete application backend -n argocd

# Port-forward cho Prometheus UI
kubectl port-forward -n dev svc/prometheus-service 9090:9090
# Mở http://localhost:9090 → Targets / Graph

# Kiểm tra targets Prometheus
kubectl exec -n dev deploy/prometheus -- wget -qO- http://localhost:9090/api/v1/targets

# Xem labels hiện có trong Loki
kubectl exec -n dev deploy/dev-payment-service -- wget -qO- "http://loki-service:3100/loki/api/v1/labels"
```

### Test API

```bash
# Health check
curl http://api.local/health

# List users (cần token)
TOKEN="eyJhbGciOiAiSFMyNTYiLCAidHlwIjogIkpXVCJ9..."
curl -H "Authorization: Bearer $TOKEN" http://api.local/api/v1/users

# List orders
curl -H "Authorization: Bearer $TOKEN" http://api.local/api/v1/orders

# Swagger
http://api.local/swagger/index.html
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
Sửa lỗi → commit → push → Argo CD tự sync lại.

### 3. App bị **Missing**

**Nguyên nhân:** Application YAML bị xoá hoặc chưa apply.

**Cách xử lý:**
```bash
kubectl apply -f apps/backend/application.yaml
```

### 4. **CrashLoopBackOff** ở Pod

**Nguyên nhân:** Bug code, thiếu ConfigMap, thiếu Secret, không kết nối được DB.

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

### 6. Không truy cập được `api.local` / `argocd.local` / `grafana.local`

**Nguyên nhân:** Thiếu host mapping hoặc ingress controller chưa enable.

**Cách xử lý:**
```bash
# Kiểm tra ingress controller
kubectl get pods -n ingress-nginx

# Kiểm tra host mapping
grep "api.local\|argocd.local\|grafana.local" /etc/hosts

# Nếu thiếu:
echo "$(minikube ip) api.local argocd.local grafana.local" | sudo tee -a /etc/hosts
```

### 7. Prune xoá mất resource

Argo CD có thể xoá resource nếu chúng không còn trong Git.

**An toàn hơn:** Tắt `prune: true` khi đang debug, hoặc set `PruneLast: true`.

### 8. Không thấy logs trong Grafana (Loki)

```bash
# Kiểm tra Fluent Bit có chạy không
kubectl get pods -n dev -l app=fluent-bit

# Kiểm tra Fluent Bit có flush log không
kubectl logs -n dev -l app=fluent-bit | grep -E "POST|200|204"

# Kiểm tra Loki labels
kubectl exec -n dev deploy/dev-payment-service -- wget -qO- "http://loki-service:3100/loki/api/v1/labels"
```

### 9. Không thấy metrics trong Grafana (Prometheus)

```bash
# Kiểm tra Prometheus targets
kubectl exec -n dev deploy/prometheus -- wget -qO- http://localhost:9090/api/v1/targets

# Kiểm tra pod có annotation prometheus không
kubectl get pods -n dev -l app=payment-service -o jsonpath='{.items[0].metadata.annotations}'
```

---

## 📐 Kiến Trúc Chi Tiết

### Sơ đồ App-of-Apps

```

    ┌─────────────────────────────────────────────────────────────┐
    │                    Root Application                          │
    │              (bootstrap/root-app.yaml)                       │
    │        Quản lý danh sách tất cả child apps                   │
    └──────────┬────────────────────────┬─────────────────────────┘
               │                        │
       ┌───────▼────────┐     ┌─────────▼──────────────────┐
       │     backend    │     │     infrastructure          │
       │ (apps/backend/)│     │ (apps/infrastructure/)      │
       │ payment-service│     │  ┌──────────────────────┐   │
       │                │     │  │ Redis                │   │
       └────────────────┘     │  │ Ingress rules        │   │
                               │  │ Prometheus + Grafana│   │
                               │  │ Loki + Fluent Bit   │   │
                               │  │ Tempo + OTel Col.   │   │
                               │  └──────────────────────┘   │
                               └─────────────────────────────┘
```

### Luồng Deploy

```
Developer
    │
    ├── [Cập nhật K8s manifests trong gitops-repo]
    │       │
    │       ▼
    ├── Push lên github.com/thuypq123/gitops-repo.git
    │       │
    │       ▼
    │   Argo CD phát hiện thay đổi (poll 3 phút)
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

### Luồng Observability

```

                         ┌──────────────────────┐
                         │    Service Code       │
                         │ (Go / Java / Python)  │
                         └─────┬─────────┬───────┘
                    ┌──────────┤         ├──────────┐
                    ▼          ▼         ▼          ▼
             ┌──────────┐ ┌──────┐ ┌────────┐ ┌──────────┐
             │ Log File  │ │ OTel │ │ OTel   │ │ Metrics  │
             │ stdout   │ │Trace │ │Metrics │ │ /metrics │
             └────┬─────┘ └──┬───┘ └───┬────┘ └────┬─────┘
                  │          │         │           │
                  ▼          ▼         ▼           ▼
             ┌──────────┐ ┌───────────────┐  ┌──────────┐
             │Flu entBit│ │ OTel Collector│  │Prometheus│
             │DaemonSet │ │               │  │ Scrape   │
             └────┬─────┘ │  └───┬───────┘  └────┬─────┘
                  │       │      │               │
                  ▼       │      ▼               ▼
             ┌────────┐   │  ┌────────┐     ┌──────────┐
             │  Loki  │   │  │ Tempo  │     │ Prometheus│
             │  Logs  │   │  │ Traces │     │  Storage  │
             └────┬───┘   │  └────────┘     └────┬─────┘
                  │       │                       │
                  └───────┼───────────────────────┘
                          │
                          ▼
                   ┌───────────┐
                   │  Grafana  │
                   │Dashboard  │
                   └───────────┘
```

### Giải thích các tham số SyncPolicy

| Tham số | Ý nghĩa |
|---------|---------|
| `prune: true` | Xoá resource trên cluster nếu không còn trong Git |
| `selfHeal: true` | Tự động sửa nếu resource bị thay đổi ngoài Git |
| `CreateNamespace=true` | Tự tạo namespace nếu chưa có |
| `PruneLast=true` | Chỉ xoá resource cũ sau khi resource mới đã ready |

---

## 💡 Mẹo & Best Practices

1. **Luôn dùng Pull Request** cho mọi thay đổi — không push thẳng vào `main`
2. **Commit message có quy tắc**: `feat:`, `fix:`, `chore:`, `infra:`
3. **Kiểm tra YAML trước khi push**:
   ```bash
   kubectl apply --dry-run=client -f apps/backend/application.yaml
   ```
4. **Mọi K8s manifest đều phải trong gitops-repo** — không đặt trong source code repo
5. **Tận dụng Kustomize patches** thay vì copy toàn bộ manifest cho mỗi môi trường
6. **Phân quyền bằng AppProject** nếu có nhiều team:
   - Mỗi team một project riêng
   - Giới hạn source repo, cluster resource được phép tạo
7. **Set `PruneLast: true`** để tránh downtime khi thay đổi workload
8. **Dùng `argocd app wait`** trong CI/CD pipeline để chờ deploy hoàn tất
9. **Backup Argo CD config** thường xuyên
10. **Không hardcode secret** trong YAML — dùng SealedSecrets hoặc External Secrets
11. **Thêm `prometheus.io/scrape: "true"` annotation** cho mọi service cần monitor
12. **Set `OTEL_SERVICE_NAME` env** cho mọi service cần tracing

---

## 🏁 Quick Start (Tóm tắt nhanh)

```bash
# 1. Start Minikube
minikube start --cpus=4 --memory=8192
minikube addons enable ingress

# 2. Cài Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Chờ Argo CD ready
kubectl wait --for=condition=ready pods --all -n argocd --timeout=300s

# 4. Lấy mật khẩu admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# 5. Thêm GitOps repo
argocd repo add https://github.com/thuypq123/gitops-repo.git \
  --username thuypq123 --password <token>

# 6. Thêm host mapping
echo "$(minikube ip) api.local argocd.local grafana.local" | sudo tee -a /etc/hosts

# 7. Bootstrap
kubectl apply -f bootstrap/root-app.yaml

# 8. Kiểm tra
kubectl get applications -n argocd

# 9. Mở Grafana
# http://grafana.local → admin / admin
# Explore → Prometheus: up{kubernetes_namespace="dev"}
# Explore → Loki: {namespace_name="dev"}
```

---

> 📝 **Tài liệu tham khảo thêm:**
> - [Argo CD Documentation](https://argo-cd.readthedocs.io/)
> - [App-of-Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
> - [Kustomize Documentation](https://kustomize.io/)
> - [Minikube Ingress Addon](https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns/)
> - [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
> - [Grafana Loki Documentation](https://grafana.com/docs/loki/)
> - [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
