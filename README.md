# Progressive Delivery, Security, and Secrets Management (Lab 1 & Lab 2)

Dự án này là chuỗi bài thực hành về **GitOps (ArgoCD)** kết hợp quản lý phân quyền (**RBAC**), chính sách an toàn tài nguyên (**OPA Gatekeeper**), quản lý bí mật bảo mật (**External Secrets Operator - ESO**) và an toàn chuỗi cung ứng phần mềm (**Trivy Scanning + Cosign Signing**).

---

## 📂 Repository Structure

```
thur-lab/
├── .github/workflows/    # CI/CD: Trivy scanning & Cosign image signing
├── app-api/              # API Rollout manifests (Argo Rollouts)
├── app-analysis/         # Analysis manifests for progressive canary validation
├── app-alert/            # Prometheus Rules cho SLO alerts
├── app-common/           # Namespace và cấu hình cơ bản
├── argocd/               # GitOps ArgoCD Apps (App of Apps pattern)
│   ├── apps/             # Các file khai báo Application (RBAC, ESO, Gatekeeper,...)
│   └── root.yaml         # Root Application quản lý toàn bộ apps
├── eso/                  # Cấu hình External Secrets Operator (AWS Secrets Manager)
├── gatekeeper/           # OPA Gatekeeper ConstraintTemplates & Constraints
│   ├── constraints/      # Các chính sách chặn tài nguyên vi phạm
│   └── templates.yaml    # Các template Rego định nghĩa luật
├── policies/             # Cấu hình Sigstore ClusterImagePolicy
├── rbac/                 # Cấu hình phân quyền Kubernetes RBAC
├── runbooks/             # Hướng dẫn xử lý sự cố (ESO Rotation & Exception ADR)
├── src/api/              # Mã nguồn Flask API & Dockerfile
└── signing/              # Chứa file Public Key (cosign.pub) dùng xác thực image
```

---

## 🔒 LAB 1: RBAC & OPA GATEKEEPER

### 1. Thành phần nộp (Deliverables)
* [rbac/](file:///d:/uni/xbrain/phase_2/w10/thur-lab/rbac/): Định nghĩa 3 Roles (Developer, SRE, Viewer) và các RoleBindings tương ứng.
* [gatekeeper/constraints/](file:///d:/uni/xbrain/phase_2/w10/thur-lab/gatekeeper/constraints/): 5 chính sách bảo mật:
  * Chặn image tag `:latest` (`disallow-latest-tag.yaml`).
  * Bắt buộc có giới hạn tài nguyên `resources.limits` (`require-resource-limits.yaml`).
  * Chặn chạy quyền `root` (`disallow-root-user.yaml`).
  * Chặn sử dụng `hostNetwork` (`disallow-host-network.yaml`).
  * Bắt buộc có nhãn `owner: <tên-người-dùng>` (`require-owner-label.yaml`).
* [argocd/apps/](file:///d:/uni/xbrain/phase_2/w10/thur-lab/argocd/apps/): Khai báo các ArgoCD Application `rbac.yaml` và `gatekeeper-policies.yaml` (có cấu hình đệ quy `directory.recurse: true`).

### 2. Cách tự kiểm tra (Self-Verification)

#### A. Kiểm tra Phân quyền RBAC
Xác nhận quyền hạn của từng Role bằng lệnh `auth can-i`:
```bash
# Developer (trong namespace demo)
kubectl auth can-i create deployments --as=system:serviceaccount:demo:developer-sa -n demo # pass
kubectl auth can-i create pods --as=system:serviceaccount:demo:developer-sa -n kube-system # reject

# SRE (toàn cluster)
kubectl auth can-i port-forward pods --as=system:serviceaccount:demo:sre-sa --all-namespaces # pass

# Viewer (toàn cluster)
kubectl auth can-i get pods --as=system:serviceaccount:demo:viewer-sa --all-namespaces # pass
kubectl auth can-i create pods --as=system:serviceaccount:demo:viewer-sa --all-namespaces # reject
```

#### B. Kiểm tra Gatekeeper Constraints
Chạy các tài nguyên vi phạm để kiểm tra tính năng chặn của Gatekeeper:
* **Chặn image :latest:**
  ```bash
  kubectl run test-latest --image=nginx:latest --dry-run=server
  # (Kỳ vọng: Bị chặn với lỗi: denied by disallow-latest-tag)
  ```
* **Chặn thiếu CPU/RAM limits:**
  ```bash
  kubectl run test-no-limits --image=nginx:1.25.1 --dry-run=server
  # (Kỳ vọng: Bị chặn với lỗi: denied by require-resource-limits)
  ```
* **Chặn chạy quyền Root:**
  ```bash
  kubectl apply -f gatekeeper/constraints/test-bad-root.yaml (hoặc deploy pod với runAsUser: 0)
  # (Kỳ vọng: Bị chặn với lỗi: denied by disallow-root-user)
  ```
* **Deployment hợp lệ (Pass):**
  ```bash
  kubectl apply -f gatekeeper/constraints/test-good.yaml (hoặc deploy Pod version pinned + limits + non-root + owner label)
  # (Kỳ vọng: Tạo thành công)
  ```

---

## 🔑 LAB 2: SECRETS MANAGEMENT & SUPPLY CHAIN SECURITY

### 1. Thành phần nộp (Deliverables)
* [eso/](file:///d:/uni/xbrain/phase_2/w10/thur-lab/eso/): `SecretStore` kết nối tới AWS Secrets Manager cục bộ và `ExternalSecret` cấu hình đồng bộ tự động mỗi `30s`.
* [signing/cosign.pub](file:///d:/uni/xbrain/phase_2/w10/thur-lab/signing/cosign.pub): Khóa công khai của Cosign (Tuyệt đối **không** commit file `.key`).
* [.github/workflows/build-push.yml](file:///d:/uni/xbrain/phase_2/w10/thur-lab/.github/workflows/build-push.yml): Pipeline CI/CD tích hợp quét bảo mật Trivy và ký số ảnh bằng Cosign.
* [argocd/apps/](file:///d:/uni/xbrain/phase_2/w10/thur-lab/argocd/apps/): Khai báo ArgoCD App `eso.yaml`, `eso-config.yaml` và `policy-controller.yaml` (có cấu hình tăng RAM/CPU cho webhook tránh CrashLoop).
* [runbooks/](file:///d:/uni/xbrain/phase_2/w10/thur-lab/runbooks/): Hướng dẫn xoay vòng secret (`eso-rotation.md`) và Exception ADR (`exception-adr.md`).

### 2. Cách tự kiểm tra (Self-Verification)

#### A. Kiểm tra xoay vòng Secret (ESO)
1. Truy cập AWS Secrets Manager, thay đổi giá trị của secret `db-password` (ví dụ sang `NewPass123!`).
2. Kiểm tra mật khẩu trong K8s Secret được đồng bộ (< 60s):
   ```bash
   kubectl get secret db-secret -n demo -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
   ```
3. Đảm bảo các pod API không bị khởi động lại:
   ```bash
   kubectl get pods -n demo
   ```
   *(Kỳ vọng: Cột RESTARTS của pod API không tăng lên, AGE giữ nguyên).*

#### B. Kiểm tra Trivy quét bảo mật
1. Thay đổi Dockerfile thành `FROM python:3.8` (chứa nhiều lỗ hổng bảo mật) và push lên GitHub.
2. Kiểm tra GitHub Actions: Pipeline sẽ chuyển sang **màu đỏ (Thất bại)** tại step Trivy Scan và chặn việc push/sign image lên GHCR.

#### C. Kiểm tra chặn Image chưa ký số (Sigstore Webhook)
Thử deploy một image bất kỳ chưa được ký lên namespace `demo`:
```bash
kubectl run test-unsigned --image=ghcr.io/dungoppai/nginx -n demo
```
*(Kỳ vọng: Bị chặn ngay lập tức bởi Admission Webhook với thông báo: admission webhook "policy.sigstore.dev" denied the request: validation failed: ... must be an image digest)*

#### D. Kiểm tra chạy Image có chữ ký hợp lệ
1. Chuyển lại Dockerfile về `FROM python:3.13-alpine` (bản sạch không có lỗi bảo mật) và push lên GitHub.
2. Pipeline sẽ chuyển sang **màu xanh (Thành công)**, tự động ký số bằng Cosign, tự động commit cập nhật tag mới vào `app-api/rollout.yaml`.
3. ArgoCD tự động đồng bộ (sync) và deploy thành công image đã ký lên cluster mà không gặp bất kỳ lỗi chặn nào.

---

## 🛠️ Quick Start (Deploy Toàn Bộ Platform)

1. Khởi động Minikube:
   ```bash
   minikube start --driver=docker
   ```
2. Cài đặt ArgoCD:
   ```bash
   kubectl create ns argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
3. Tạo AWS credentials secret phục vụ cho ESO (chạy thủ công trực tiếp, không lưu vào git):
   ```bash
   kubectl create secret generic awssm-secret -n demo \
     --from-literal=access-key-id="<YOUR_ACCESS_KEY>" \
     --from-literal=secret-access-key="<YOUR_SECRET_KEY>"
   ```
4. Áp dụng Root Application (App of Apps) để cài đặt toàn bộ hệ thống tự động:
   ```bash
   kubectl apply -f argocd/root.yaml
   ```
   *(Nhờ cơ chế sync-wave được thiết lập tỉ mỉ, các Operators và CRD sẽ tự động cài đặt theo đúng thứ tự mà không bị lỗi chéo).*
