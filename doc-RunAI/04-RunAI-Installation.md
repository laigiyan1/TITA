# NVIDIA Run:ai 安裝流程

> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/preparations  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/install-control-plane  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/helm-install  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation  
> 整理日期：2026-05-20  
> 目標版本：NVIDIA Run:ai v2.25  
> **部署前請至 [官方安裝文件](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation) 確認最新版本**

---

## 1. 安裝架構說明

本案採用 **Self-hosted + Helm 安裝**模式，共需安裝兩個元件：

```
安裝順序：
1. Run:ai Control Plane（命名空間：runai-backend）
2. Run:ai Cluster（命名空間：runai）

兩者均安裝於同一個 Kubernetes 叢集
（R670 節點管理 Control Plane；XE9780 節點承載 Cluster 的 GPU 調度服務）
```

---

## 2. 安裝前準備

### 2.1 NVIDIA NGC 帳號與 API Key

1. 前往 [NVIDIA NGC](https://ngc.nvidia.com/) 建立帳號
2. 建立 API Key：Profile → Setup → Generate API Key
3. 記錄 API Key（格式：`nvapi-xxxxxxxxxxxx`）

### 2.2 確認先決條件

執行以下指令確認環境就緒：

```bash
# K8s 叢集狀態
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# Ingress Controller
kubectl get pods -n ingress-nginx

# Prometheus
kubectl get pods -n monitoring

# GPU Operator
kubectl get pods -n gpu-operator

# Default Storage Class
kubectl get storageclass | grep default

# Helm 版本（需 >= 3.14）
helm version
```

### 2.3 FQDN 與 TLS 準備

Run:ai 需要可解析的 FQDN，**不得使用純 IP**：

| 用途 | 範例 FQDN |
|------|-----------|
| Control Plane 入口 | `runai.example.internal` |
| Workspace / 工作負載子網域 | `*.runai.example.internal` |

> ⚠️ **v2.25 變更：Host-based Routing 預設啟用（Kubernetes 環境）**  
> Run:ai v2.25 起，Kubernetes 叢集的 Host-based Routing **預設開啟**（支援 RStudio、VS Code 等工具）。工作負載端點以子網域形式呈現（如 `<workload-name>.runai.example.internal`）。  
> **本案情境（Kubernetes + 預設 Host-based Routing）下，部署前應準備**：
> 1. 萬用字元 DNS 記錄：`*.runai.example.internal` → Ingress Controller IP
> 2. 萬用字元 TLS 憑證：`*.runai.example.internal`
>
> 例外情形：若停用 Host-based Routing 改用 **path-based routing**，Workspace/Training 子網域 DNS 與萬用字元憑證**不需要**；OpenShift 環境另有不同要求。  
> 來源：[What's New v2.25 – Host-based Routing](https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25) | [Cluster System Requirements](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/system-requirements)

```bash
# 確認 FQDN 可解析
nslookup runai.example.internal
# 確認萬用字元解析（以任意子網域測試）
nslookup test.runai.example.internal
```

---

## 3. 安裝 Control Plane

### 步驟 1：新增 Helm Repo（連線環境）

```bash
NGC_API_KEY="<your-ngc-api-key>"   # 替換為實際 API Key

helm repo add runai https://helm.ngc.nvidia.com/nvidia/runai \
  --force-update \
  --username='$oauthtoken' \
  --password=${NGC_API_KEY}
helm repo update
```

### 步驟 2：建立命名空間與映像拉取 Secret

```bash
# 建立命名空間
kubectl create namespace runai-backend

# 建立映像拉取憑證
kubectl create secret docker-registry runai-reg-creds \
  --docker-server=https://nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password=${NGC_API_KEY} \
  --docker-email=<your-email@example.com> \
  --namespace=runai-backend
```

### 步驟 3：安裝 Control Plane

#### 標準安裝

```bash
RUNAI_DOMAIN="runai.example.internal"  # 替換為實際 FQDN
ADMIN_EMAIL="admin@example.com"         # 替換為實際管理員 Email
ADMIN_PASSWORD="<SecureP@ssw0rd>"       # 最少 8 字元，含大小寫、數字、特殊字元

helm upgrade -i runai-backend -n runai-backend runai/control-plane \
  --set global.domain=${RUNAI_DOMAIN} \
  --set global.ingress.ingressClass=haproxy \
  --set tenantsManager.config.adminUsername=${ADMIN_EMAIL} \
  --set tenantsManager.config.adminPassword="${ADMIN_PASSWORD}"
```

> **Ingress Class 說明**：官方文件範例使用 `haproxy`（建議選項）。  
> 若使用 NGINX：`--set global.ingress.ingressClass=nginx`  
> 若使用 Rancher NGINX（RKE2 內建）：`--set global.ingress.ingressClass=nginx`  
> `ingressClass` 的值必須對應叢集中實際安裝的 Ingress Controller IngressClass 名稱。  
> 若使用自訂 CA 憑證：加上 `--set global.customCA.enabled=true`

#### Air-gapped 安裝（離線環境）

```bash
# 從 NGC 下載離線套件（在可連網機器上執行）
RUNAI_VERSION="2.25.x"   # 替換為實際版本號
curl -LO --request GET \
  "https://api.ngc.nvidia.com/v2/org/nvidia/team/runai/resources/runai-airgapp-package/versions/${RUNAI_VERSION}/files/runai-airgapped-package-${RUNAI_VERSION}.tar.gz" \
  -H "Authorization: Bearer ${NGC_API_KEY}" \
  -H "Content-Type: application/json"

# 解壓並傳輸至隔離環境（需 20 GB 以上磁碟空間）
tar -xzf runai-airgapped-package-${RUNAI_VERSION}.tar.gz

# 在隔離環境安裝
helm upgrade -i runai-backend ./chart/control-plane-${RUNAI_VERSION}.tgz \
    --set global.domain=${RUNAI_DOMAIN} \
    --set global.customCA.enabled=true \
    --set global.ingress.ingressClass=haproxy \
    --set tenantsManager.config.adminUsername=${ADMIN_EMAIL} \
    --set tenantsManager.config.adminPassword="${ADMIN_PASSWORD}" \
    -n runai-backend -f custom-env.yaml
```

### 步驟 4：等待 Control Plane 就緒

```bash
# 監控 Pod 狀態（等待所有 Pod Running）
kubectl get pods -n runai-backend -w

# Control Plane 初始化最多需 15 分鐘
# 確認所有 Deployment 就緒
kubectl get deployment -n runai-backend
```

**期望輸出**（DESIRED == AVAILABLE）：
```
NAME                         READY   UP-TO-DATE   AVAILABLE
runai-backend-...            1/1     1            1
...
```

### 步驟 5：驗證 Control Plane 存取

```bash
# 確認 Ingress 建立
kubectl get ingress -n runai-backend
```

瀏覽器訪問：`https://runai.<DOMAIN>.internal`

使用步驟 3 設定的管理員帳號登入，**首次登入後請立即修改密碼**。

---

## 4. 安裝 Run:ai Cluster

### 步驟 1：完成 Control Plane Onboarding Wizard

1. 登入 Run:ai UI（`https://runai.<DOMAIN>`）
2. **Onboarding Wizard 會自動開啟**（請勿在完成前關閉視窗）
3. 填入：
   - **Cluster 名稱**：任意識別名稱（例如 `xe9780-cluster`）
   - **Cluster 位置**：選擇「Same as control plane」（本案 Control Plane 與 Cluster 同一 K8s 叢集）
4. Wizard 會生成 Helm 安裝指令（含 `controlPlane.url`、`controlPlane.clientSecret`、`cluster.uid`、`cluster.url`）

### 步驟 2：建立命名空間與映像拉取 Secret

```bash
# 先手動建立命名空間（Secret 建立需要命名空間已存在）
kubectl create namespace runai

# 建立映像拉取憑證
kubectl create secret docker-registry runai-reg-creds \
  --docker-server=https://nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password=${NGC_API_KEY} \
  --docker-email=<your-email@example.com> \
  --namespace=runai
```

> **注意**：`--create-namespace` 在 `helm upgrade -i` 時才會建立命名空間，但 Secret 建立指令需要命名空間已存在，因此必須先手動建立。

### 步驟 3：安裝 Cluster

使用 Onboarding Wizard 生成的 Helm 指令（以下為格式範例）：

#### 連線環境

```bash
helm upgrade -i runai-cluster runai/runai-cluster -n runai \
    --set controlPlane.url=<CONTROL_PLANE_URL> \
    --set controlPlane.clientSecret=<CLIENT_SECRET> \
    --set cluster.uid=<CLUSTER_UID> \
    --set cluster.url=<CLUSTER_URL> \
    --version="<RUNAI_VERSION>" \
    --create-namespace \
    --set clusterConfig.global.ingress.ingressClass=haproxy
```

> **重要**：`controlPlane.url`、`controlPlane.clientSecret`、`cluster.uid`、`cluster.url` 的值**必須從 Onboarding Wizard 複製**，不得手動填寫。  
> `ingressClass` 須與叢集實際安裝的 Ingress Controller 一致（haproxy / nginx / 其他）。

#### Air-gapped 環境

```bash
helm upgrade -i runai-cluster ./chart/runai-cluster-<VERSION>.tgz \
    --set controlPlane.url=<CONTROL_PLANE_URL> \
    --set controlPlane.clientSecret=<CLIENT_SECRET> \
    --set cluster.uid=<CLUSTER_UID> \
    --set cluster.url=<CLUSTER_URL> \
    --create-namespace \
    --set global.image.registry=<INTERNAL_REGISTRY> \
    --set global.customCA.enabled=true \
    --set clusterConfig.global.ingress.ingressClass=haproxy
```

### 步驟 4：確認 Cluster 連線

```bash
# 監控 Cluster Pod 狀態
kubectl get pods -n runai -w
```

Run:ai UI 應顯示：
1. 安裝中：「Waiting for cluster to connect」
2. 成功：「Cluster connected」（Cluster 顯示於 Clusters 表格）

---

## 5. 安裝後確認清單

```bash
# Control Plane Pod 狀態
kubectl get pods -n runai-backend

# Cluster Pod 狀態
kubectl get pods -n runai

# Cluster Sync 狀態
kubectl get pod -n runai | grep cluster-sync

# Agent 狀態
kubectl get pod -n runai | grep agent

# Deployment 狀態（DESIRED == AVAILABLE）
kubectl get deployment -n runai
kubectl get deployment -n runai-backend

# StatefulSet（資料庫元件）
kubectl get statefulset -n runai-backend
```

---

## 6. 常見問題

| 問題 | 可能原因 | 處理方式 |
|------|----------|----------|
| Control Plane Pod CrashLoopBackOff | 儲存類別未設定 | 確認 Default Storage Class 存在 |
| Ingress 無法存取 | Ingress Controller 未就緒 | 確認 Ingress Controller Pod 狀態 |
| Cluster 一直顯示「Waiting to connect」 | FQDN 解析問題 | 確認 DNS 可解析 Control Plane FQDN |
| 映像無法拉取（ImagePullBackOff） | Secret 設定錯誤 | 確認 `runai-reg-creds` Secret 正確 |
| Admin 密碼格式錯誤 | 密碼不符合複雜度要求 | 需含大小寫、數字、特殊字元，最少 8 碼 |

---

## 7. 下一步

安裝完成後，進行系統設定：

→ [05-Post-Install-Configuration.md](05-Post-Install-Configuration.md)
