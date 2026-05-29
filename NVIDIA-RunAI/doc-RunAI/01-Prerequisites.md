# NVIDIA Run:ai 先決條件與系統需求

> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/system-requirements  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/cp-system-requirements  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/network-requirements  
> 整理日期：2026-05-20  
> 目標版本：NVIDIA Run:ai v2.25  
> **部署前請至 [Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix) 確認最新版本**

---

## 1. Kubernetes 版本支援矩陣

### 1.1 Run:ai 版本與 Kubernetes 版本對應

| Run:ai 版本 | Kubernetes 版本 | OpenShift 版本 |
|-------------|-----------------|-----------------|
| **2.25（最新）** | **1.33 – 1.35** | **4.18 – 4.21** |
| 2.24 | 1.33 – 1.35 | 4.17 – 4.20 |
| 2.23 | 1.31 – 1.34 | 4.16 – 4.19 |
| 2.22 | 1.31 – 1.33 | 4.15 – 4.19 |

> 來源：[Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix)

### 1.2 支援的 Kubernetes 發行版

| 發行版 | 說明 |
|--------|------|
| Vanilla Kubernetes（kubeadm） | ✅ 支援 |
| RKE2 | ✅ 支援（含預設 Ingress Controller） |
| OpenShift Container Platform | ✅ 支援（含預設 Ingress、Container Runtime） |
| EKS（AWS） | ✅ 支援；**不支援 Bottlerocket 或 Amazon Linux** |
| GKE（Google Cloud） | ✅ 支援 Ubuntu 或 COS（COS 需 GPU Operator 24.6+） |
| AKS（Azure） | ✅ 支援 |
| OKE（Oracle） | ✅ 支援（**僅限 Ubuntu**） |

> 本案採用 Bare Metal + Ubuntu 22.04，建議選擇 Vanilla Kubernetes（kubeadm）或 RKE2。  
> ⚠️ **Dell AI Factory orchestration 參考路徑**：Dell 官方 2-8-9-400 brief（XE9680 參考平台）所列的 Kubernetes 發行版為 **Upstream Kubernetes 或 Red Hat OpenShift Container Platform**；本文件採用 RKE2 / kubeadm 屬本案部署選型，若需完整對齊 Dell AI Factory endorsed stack，請由 Dell / NVIDIA SE 確認。

---

## 2. 作業系統需求

- **需求**：任何 **Kubernetes 與 NVIDIA GPU Operator 均支援**的 Linux 發行版
- **官方測試環境**：Ubuntu 22.04 LTS、Ubuntu 24.04 LTS（Kubernetes 部署）、CoreOS（OpenShift）
- **架構支援**：x86（推薦）、ARM
- **本案建議**：

| 節點 | 建議 OS | 說明 |
|------|---------|------|
| **Dell XE9780**（GPU Worker） | **Ubuntu 24.04 LTS** ✅ | Dell 官方 Ubuntu 認證版本（Canonical Cert ID: 202510-37982）；22.04 未列入 XE9780 認證範圍 |
| **Dell R670**（管理節點） | Ubuntu 22.04 LTS 或 24.04 LTS | 兩者均在 GPU Operator 26.3.x 支援矩陣內 |

> ⚠️ Dell PowerEdge XE9780 的 Ubuntu 官方認證（Canonical 認證資料庫）僅涵蓋 **Ubuntu 24.04 LTS**。使用 Ubuntu 22.04 LTS 在 XE9780 上，GPU Operator 26.3.x 技術上仍支援，但若發生硬體相容性問題，Dell Support 可能以非認證 OS 為由拒絕協助。

---

## 3. 硬體需求

### 3.1 Control Plane 節點（R670 承載）

| 項目 | 最低需求 |
|------|----------|
| CPU | **10 核心** |
| 記憶體 | **12 GB RAM** |
| 磁碟空間 | **110 GB** |

> 來源：[Control Plane 系統需求](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/cp-system-requirements)

### 3.2 Cluster 系統節點（管理節點，非 GPU Worker）

| 項目 | 最低需求 |
|------|----------|
| CPU | **10 核心** |
| 記憶體 | **20 GB RAM** |
| 磁碟空間 | **50 GB** |

### 3.3 Worker 節點（XE9780 GPU 節點）

| 項目 | 最低需求（每節點） |
|------|------------------|
| CPU | **2 核心** |
| 記憶體 | **4 GB RAM** |

> GPU 本身需符合 NVIDIA GPU Operator 支援清單（見第 5 節）

### 3.4 安裝執行機器需求

安裝腳本執行機器（通常為 K8s Master）需具備：

| 項目 | 需求 |
|------|------|
| 磁碟空間 | **至少 50 GB 可用空間** |
| Docker | 已安裝 |
| Helm | **3.14 或以上版本** |

---

## 4. 軟體元件版本矩陣（Run:ai v2.25）

### 4.1 必要元件

| 元件 | 版本範圍 | 說明 |
|------|----------|------|
| **NVIDIA GPU Operator** | **25.10 – 26.3** ⚠️ | **必須安裝**，管理 GPU 節點；**建議選用 26.3.x**（25.10.x 在 GPU Operator 生命週期中已列為 Deprecated） |
| **Prometheus / Kube-Prometheus Stack** | **3.5 / 76.0+** | **必須安裝**，Run:ai 依賴此做指標收集 |
| **Ingress Controller** | 任意相容版本 | **必須安裝**（RKE2 內建；kubeadm 需另裝） |
| **Helm** | **3.14+** | 安裝工具 |
| **kubectl** | 相容 K8s 版本 | 叢集操作工具 |

### 4.2 選用元件（依功能需求）

| 元件 | 版本 | 用途 |
|------|------|------|
| NVIDIA Network Operator | 25.10 – 26.1 | RDMA / NVLink 多節點工作負載 |
| NVIDIA DRA Driver | 25.8 – 25.12 | Multi-Node NVLink 部署（GB200 等）；What's New v2.25 另提及 25.15，但 Support Matrix 列 25.8–25.12，需向 NVIDIA/NGC chart 驗證後才能確認 25.15 是否正式支援 |
| Knative Serving | 1.19 – 1.21 | 推論服務（Inference） |
| Kubeflow Training Operator | v1.9.2 | 分散式訓練 |
| MPI Operator | v0.6.0+ | MPI 工作負載 |
| Leader Worker Set (LWS) | v0.7.0+ | 分散式推論 |
| MetalLB | 任意版本 | 裸機環境 LoadBalancer（建議） |

> 來源：[Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix)

> ⚠️ **GPU Operator 版本補充說明（官方文件內部不一致）**：  
> - 官方 Support Matrix **表格**列 GPU Operator 版本範圍 **25.10–26.3**  
> - 同頁 Supported NVIDIA GPUs 段落仍有 **25.3–25.10** 的舊文字  
> - [NVIDIA GPU Operator 生命週期政策](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html) 明確標示：**25.10.x = Deprecated**、**26.3.x = Current Supported**  
> - **建議**：選用 26.3.x，部署前以 Run:ai v2.25 Support Matrix 與實際 Helm chart 測試確認  
> 來源：[Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix) | [GPU Operator Platform Support](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html)

---

## 5. GPU 需求

### 5.1 支援的 GPU 類型

- **支援**：所有 NVIDIA 資料中心 GPU（由 NVIDIA GPU Operator 支援者皆相容）
- **GPU 架構**：T、V、A、L、H、B、GH 世代

### 5.2 不支援的硬體

| 不支援項目 | 原因 |
|-----------|------|
| NVIDIA DGX Spark | 不在支援清單 |
| NVIDIA Jetson 系列 | 嵌入式平台，不支援 |
| 工作站 GPU（GeForce / RTX） | 非資料中心 GPU |
| vGPU（虛擬 GPU） | **不支援**；僅支援 GPU Passthrough |

> **本案 Dell XE9780 使用的 GPU 型號請對照 [GPU Operator 支援矩陣](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html) 確認相容性**

---

## 6. 網路需求

> 來源：[網路需求](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/network-requirements)

### 6.1 入站需求（Inbound）

| 連接埠 | 協定 | 來源 | 用途 |
|--------|------|------|------|
| 443 | HTTPS | 0.0.0.0（任意） | Web UI、CLI、API 存取 |

### 6.2 出站需求（Outbound）

#### 必要出站連線

| 目標 | 連接埠 | 用途 |
|------|--------|------|
| Control Plane FQDN | 443 | Cluster ↔ Control Plane 同步 |
| `nvcr.io` | 443 | 拉取 Container Images 與 Helm Charts |
| `api.ngc.nvidia.com` | 443 | NGC 目錄瀏覽 |
| `huggingface.co` | 443 | 模型瀏覽（選用） |

#### 選用出站連線（依安裝元件）

| 目標 | 連接埠 | 用途 |
|------|--------|------|
| `docker.io` | 443 | HAProxy、Training Operator 映像檔 |
| `gcr.io` | 443 | GPU Operator、Knative 映像檔 |
| `quay.io` | 443 | Prometheus Operator 映像檔 |

> **Air-gapped 環境**：若無法連線上述 URL，須提前下載所有映像檔至內部 Registry（需至少 20 GB 磁碟空間）

### 6.3 內部網路需求

| 需求 | 說明 |
|------|------|
| K8s 節點互通 | **所有節點須能全面互通**（Kubernetes 架構要求） |
| NTP 時間同步 | **必須**，所有節點須以 NTP 同步時間 |

### 6.4 IPv6 限制

- `nvcr.io` 與 `runai.jfrog.io` **僅有 IPv4 DNS 記錄**
- IPv6-only 環境需實作 NAT64/DNS64 轉換，或部署內部 Mirror Registry

---

## 7. 儲存與憑證需求

### 7.1 儲存類別（Storage Class）

| 需求 | 說明 |
|------|------|
| Default Storage Class | K8s 叢集必須設有 Default Storage Class（供 Control Plane 使用） |
| 共享儲存 | 工作節點之間的資料集存取需 NFS 或 NAS 協定（AI 工作負載資料共享） |

#### Dell AI Factory 推薦儲存組合

Dell AI Factory 生態系中，常見的儲存搭配如下（依功能用途選擇，並非強制要求）：

| 儲存方案 | 協定 | 適用場景 | 備注 |
|----------|------|----------|------|
| **Dell PowerScale（Isilon）** | NFS / SMB | 訓練資料集、模型快取（RWX 共享儲存） | Dell AI Factory 主流 NAS 選擇 |
| **Dell PowerStore** | iSCSI / FC / NVMe-oF | 高效能區塊儲存（資料庫、PostgreSQL PVC） | 低延遲場景 |
| **Dell ObjectScale** | S3-compatible | 模型倉庫、Checkpoint 長期存放 | 大容量非結構化資料 |
| 通用 NFS（本案小型環境） | NFS v4 | 訓練資料 / PVC 共享 | 可使用 R670 上的 NFS Server 作為入門方案 |

> **本案小型環境建議**：若未採購獨立 Dell 儲存設備，可在 R670 上以 NFS Server 提供共享儲存（適合 PoC / 初期建置）。  
> ⚠️ **Dell AI Factory 參考配置儲存說明**：Dell 官方 2-8-9-400 brief（XE9680 參考平台）搭配的儲存設備為 **Dell PowerScale F710**（NFS 共享儲存）。本案 R670 上的 NFS Server 為**入門 PoC 方案**，非 Dell brief 驗證路徑；正式生產環境若需對齊 Dell AI Factory 推薦配置，請依採購 BOM 確認儲存型號，並由 Dell / NVIDIA SE 協助驗證。

### 7.2 TLS 憑證需求

> ⚠️ **v2.25 變更：Host-based Routing 預設啟用（Kubernetes 環境）**  
> Run:ai v2.25 起，Kubernetes 叢集的 Host-based Routing **預設開啟**（支援 RStudio、VS Code 等工具），工作負載 URL 以子網域形式呈現（如 `<workload>.<DOMAIN>`）。  
> **本案情境（Kubernetes + 預設 Host-based Routing）下，萬用字元 DNS 與 TLS 憑證為必要**。  
> 例外情形：
> - 若停用 Host-based Routing 改用 **path-based routing**，Workspace/Training 萬用字元憑證**不需要**
> - **OpenShift** 環境的路由需求與 Kubernetes 不同，需另參閱官方 OpenShift 部署說明
>
> 來源：[What's New v2.25 – Host-based Routing](https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25) | [Cluster System Requirements](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/system-requirements)

Run:ai 要求三類 TLS 憑證：

| 憑證用途 | 說明 |
|----------|------|
| Control Plane 網域憑證 | `runai.<DOMAIN>` 的 HTTPS 憑證（必要） |
| Workspace / Training 萬用字元憑證 | `*.<DOMAIN>`（Kubernetes + 預設 Host-based Routing 時必要；改用 path-based routing 時不需要） |
| Inference 萬用字元憑證 | `*.<inference-domain>` |

### 7.3 FQDN 需求

- Control Plane 需設定可解析的 FQDN（**不得使用純 IP 位址**）
- 多集群部署時，Cluster 亦需獨立 FQDN
- DNS 必須能從所有節點解析

---

## 8. 授權需求

| 項目 | 說明 |
|------|------|
| NVIDIA NGC 帳號 | 必須，用於拉取 Run:ai 映像檔與 Helm Charts |
| NGC API Key | 安裝前需建立，用於 Helm 認證與映像檔拉取 |
| NVIDIA AI Enterprise 授權 | Run:ai v2.20 後整合於 NVAIE（請向 NVIDIA 業務確認授權方式） |

---

## 9. 先決條件安裝順序

部署 Run:ai 前，K8s 叢集上須依序完成以下元件安裝：

```
1. Kubernetes 叢集建置（kubeadm / RKE2）
2. Default Storage Class 設定
3. Ingress Controller 安裝（kubeadm 環境）
4. MetalLB 安裝（Bare Metal LoadBalancer）
5. Prometheus / Kube-Prometheus Stack 安裝
6. NVIDIA GPU Operator 安裝（GPU Worker 節點）
7. TLS 憑證配置
8. DNS / FQDN 設定
9. NTP 時間同步確認
```

> 詳細步驟見 [02-Kubernetes-Platform-Setup.md](02-Kubernetes-Platform-Setup.md)、[03-NVIDIA-GPU-Operator-Setup.md](03-NVIDIA-GPU-Operator-Setup.md)
