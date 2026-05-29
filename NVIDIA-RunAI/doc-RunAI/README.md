# NVIDIA Run:ai 技術文件庫

> 整理日期：2026-05-23（含 NVAIE 架構補充、NVIDIA ERA 參考架構對齊）  
> 目標版本：NVIDIA Run:ai v2.25（最新 Active 版本）  
> 建置情境：Dell PowerEdge XE9780（GPU 節點）× 1 + Dell PowerEdge R670（管理節點，無 GPU）× 2，以 Bare Metal K8s + Self-hosted 模式部署  
> **方案框架：參考 NVIDIA Enterprise Reference Architecture 2-8-9-400 HGX 設計模式（實際 Dell AI Factory endorsement 需依 Dell / NVIDIA 確認）**

---

## Dell AI Factory / NVIDIA ERA 參考架構對齊

本案 GPU 節點採用 **Dell PowerEdge XE9780**，其 8-GPU HGX 設計與 NVIDIA Enterprise Reference Architecture（ERA）2-8-9-400 的 HGX 節點設計模式高度對應，適合作為 Dell AI Factory 定位的企業 AI 基礎架構。

> ⚠️ **重要說明**：NVIDIA ERA White Paper 說明的是通用 HGX 2-8-9-400 設計模式（非 Dell 專屬）；Dell 官方 2-8-9-400 brief 及 NVIDIA ERA index 中明確列出的 Dell 參考平台為 **Dell PowerEdge XE9680**，非本案採購的 XE9780。因此本案的正確描述方式為「**參考 / 對齊 NVIDIA ERA 2-8-9-400 HGX 設計模式**」，而非「已獲 Dell AI Factory ERA endorsement 覆蓋」。XE9780 是否屬於正式 endorsed configuration，**需透過 Dell / NVIDIA SE、正式 BOM 或最新 Dell 官方文件確認**。

```
NVIDIA ERA 2-8-9-400 命名規則（描述單台 HGX GPU 伺服器規格）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  2   → 每台伺服器的 CPU 數量             → 2× CPU
  8   → 每台伺服器的 GPU 數量             → 8× GPU（HGX B200 SXM6 / B300 NVL8）
  9   → 每台伺服器的 NIC 數量             → 9 張網卡：
          8× B3140H SuperNIC（東西向，GPU Compute E-W）
          1× B3220 DPU（南北向，Storage / Management N-S）
  400 → 每張東西向 NIC 的頻寬（Gb/s）    → 400Gb/s × 8 NIC = 3.2Tb/s 東西向總頻寬

  ERA 2-8-9-400 briefed reference platform：Dell PowerEdge XE9680
  本案實際 GPU 節點：Dell PowerEdge XE9780（endorsement 待確認）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

> 參照圖片：[era-2-8-9-400-single-server.png](../pic-RunAI/era-2-8-9-400-single-server.png)（NVIDIA ERA White Paper 通用 HGX 節點示意圖，非 XE9780 或 XE9680 專屬）

```
方案堆疊（本案三台設備對應）：
  Application Layer    → NIM、NeMo、PyTorch / TF（NVAIE 軟體）
  Orchestration Layer  → NVIDIA Run:ai v2.25（本文件庫核心）
  Infrastructure Layer → NVIDIA GPU Operator + Network Operator（NVAIE）
  Platform Layer       → Kubernetes（RKE2 / kubeadm）+ Ubuntu 24.04（XE9780 建議）/ 22.04（R670 可）
  Hardware Layer       → Dell PowerEdge XE9780 × 1（參考 2-8-9-400 HGX 節點設計，
                           BOM / endorsement 待 Dell / NVIDIA 確認）
                         Dell PowerEdge R670 × 2（本案管理節點；
                           Dell 2-8-9-400 brief 參考管理節點數為 4 台 R670，
                           本案為縮減部署）
```

> **NVIDIA ERA 參考資料**：[ERA White Paper — Key Building Blocks](https://docs.nvidia.com/enterprise-reference-architectures/white-paper/latest/key-building-blocks.html)　｜　[Dell AI Factory 2-8-9-400 Brief（XE9680 參考平台）](https://www.delltechnologies.com/asset/en-us/solutions/infrastructure-solutions/briefs-summaries/nvidia-2-8-9-400-configuration-era-endorsed-for-the-dell-ai-factory-with-nvidia-brief.pdf)

> ⚠️ **GPU 型號說明**：依 Canonical Ubuntu 認證資料庫（Cert ID: 202510-37982），Dell PowerEdge XE9780 認證測試的 GPU 為 **Dell HGX B200 180GB SXM6** 或 **Dell HGX B300 NVL8 270GB SXM6**。最終型號請依採購 BOM / Dell Service Tag 確認；確認後更新各文件中驅動版本、MIG Profile 等對應欄位。

---

## 文件清單

| 檔案 | 說明 |
|------|------|
| [00-Overview-and-Architecture.md](00-Overview-and-Architecture.md) | Run:ai 產品概覽、架構元件、Self-hosted vs SaaS 說明 |
| [01-Prerequisites.md](01-Prerequisites.md) | 完整先決條件：K8s 版本、OS、GPU、軟體元件版本矩陣 |
| [02-Kubernetes-Platform-Setup.md](02-Kubernetes-Platform-Setup.md) | K8s 平台選型（RKE2 / kubeadm）及基礎建置指引 |
| [03-NVIDIA-GPU-Operator-Setup.md](03-NVIDIA-GPU-Operator-Setup.md) | NVIDIA GPU Operator 安裝（GPU 節點先決條件） |
| [04-RunAI-Installation.md](04-RunAI-Installation.md) | Run:ai Control Plane + Cluster 完整安裝流程 |
| [05-Post-Install-Configuration.md](05-Post-Install-Configuration.md) | 安裝後設定：SSO、Email、專案、授權 |
| [06-Validation-and-Monitoring.md](06-Validation-and-Monitoring.md) | 驗證與監控：健康檢查、告警、疑難排解 |
| [07-Deployment-Checklist.md](07-Deployment-Checklist.md) | 逐步部署清單（含硬體、K8s、GPU Operator、Run:ai） |
| [08-NVAIE-Overview.md](08-NVAIE-Overview.md) | **NVAIE 平台概覽**：兩層可組合式架構、元件清單、版本矩陣、授權模式 |
| [09-NVAIE-RunAI-Integration.md](09-NVAIE-RunAI-Integration.md) | **NVAIE × Run:ai 整合架構**：Run:ai 在 NVAIE 中的角色、部署模式比較、授權說明 |

---

## 硬體環境對應角色

| 設備 | 規格 | 角色 |
|------|------|------|
| Dell XE9780 × 1 | 含 NVIDIA GPU | K8s GPU Worker Node（執行 AI 工作負載） |
| Dell R670 × 2 | 無 GPU | K8s Control Plane / Run:ai Control Plane（管理節點） |

> **建置說明**：R670 × 2 作為 K8s Master 節點，同時承載 Run:ai Control Plane；XE9780 × 1 作為 K8s Worker Node，Run:ai Cluster 服務部署於此節點以管理 GPU 資源。
>
> ⚠️ **HA 注意事項**：本案 2 台管理節點可承載 Kubernetes 管理服務，但 **2 個 etcd member 並不構成標準 HA quorum**（Kubernetes 官方建議使用奇數個 control plane / etcd member，例如 3 台）。若任一 R670 故障，stacked etcd 可能失去 quorum 導致 Control Plane 不可用。如需生產級 HA，建議評估以下方案：
> - 擴充至 **3 台** K8s Control Plane 節點（stacked etcd），或
> - 採用**外部 etcd 叢集**（external etcd）+ 獨立的 HA datastore 設計。
>
> 來源：[Kubernetes HA 架構](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)、[etcd 叢集設計建議](https://etcd.io/docs/current/op-guide/clustering/)

---

## 建議閱讀路徑

### 路徑 A：首次了解 NVIDIA 生態系（概念優先）⭐ 建議新人
```
08（NVAIE 是什麼）→ 09（NVAIE × Run:ai 整合關係）→ 00（Run:ai 架構細節）→ 07（部署清單預覽）
```

### 路徑 B：完整部署（Infrastructure Admin）
```
00（架構）→ 01（先決條件）→ 02（K8s 平台建置）→ 
03（GPU Operator）→ 04（Run:ai 安裝）→ 
05（安裝後設定）→ 06（驗證）→ 07（逐步清單）
```

### 路徑 C：快速確認先決條件
```
01（先決條件）→ 07（部署清單）
```

### 路徑 D：了解 NVAIE 授權與 Run:ai 關係（採購 / 規劃人員）
```
08（NVAIE 平台與授權）→ 09（Run:ai 在 NVAIE 的授權與部署模式）
```

---

## K8s 平台選型決策

| 選項 | 適用場景 | 優點 | 注意事項 |
|------|----------|------|----------|
| **RKE2**（Recommended） | 企業 Bare Metal | 安全性佳、含預設 Ingress（NGINX）、官方支援 | 需要學習 RKE2 CLI |
| **kubeadm** | 標準 Kubernetes | 最接近 vanilla K8s、社群資源豐富 | 需手動安裝 Ingress Controller |

> **注意**：k3s **不在** Run:ai 官方 Support Matrix 的支援發行版清單中，不列為本案部署選項。  
> 來源：[Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix)

> 詳見 [02-Kubernetes-Platform-Setup.md](02-Kubernetes-Platform-Setup.md)

---

## 版本選擇決策

- **Run:ai v2.25**：目前最新 Active 版本（截至 2026-05）；What's New 已發布，部署版本仍需以 Support Matrix 與 NGC 可取得版本為準  
  來源：[What's New v2.25](https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25)
- **部署前請至 [Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix) 確認最新版本**
- 舊版 v2.19、v2.20 已轉移至新文件平台，部分頁面不再可訪問

---

## 架構圖（pic-RunAI 資料夾）

| 檔案 | 說明 |
|------|------|
| [nvaie-composable-stack.svg](../pic-RunAI/nvaie-composable-stack.svg) | NVAIE 兩層可組合式架構（官方 SVG） |
| [nvaie-infra-stack-alignment.svg](../pic-RunAI/nvaie-infra-stack-alignment.svg) | NVAIE Infrastructure 版本分支對齊圖 |
| [nvaie-platform-overview.png](../pic-RunAI/nvaie-platform-overview.png) | NVAIE 平台概覽圖（Reference Architecture） |
| [nvaie-software-stack.png](../pic-RunAI/nvaie-software-stack.png) | NVAIE 軟體堆疊分層圖 |

---

## 主要官方參考 URL

### Dell AI Factory × NVIDIA Reference Architecture

| 用途 | URL |
|------|-----|
| **NVIDIA Enterprise Reference Architectures 入口** | https://docs.nvidia.com/enterprise-reference-architectures/index.html |
| **NVIDIA HGX AI Factory RA（本案主要參考架構）** | https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/index.html |
| **Dell AI Factory 2-8-9-400 Brief（XE9680 參考平台）** | https://www.delltechnologies.com/asset/en-us/solutions/infrastructure-solutions/briefs-summaries/nvidia-2-8-9-400-configuration-era-endorsed-for-the-dell-ai-factory-with-nvidia-brief.pdf |

### NVAIE 官方文件

| 用途 | URL |
|------|-----|
| **NVAIE 官方文件入口** | https://docs.nvidia.com/ai-enterprise/ |
| NVAIE 8.1 Release Notes | https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html |
| NVAIE 軟體元件清單 | https://docs.nvidia.com/ai-enterprise/software/latest/overview.html |
| NVAIE Infrastructure 生命週期 | https://docs.nvidia.com/ai-enterprise/lifecycle/latest/infrastructure-software.html |
| NVAIE 授權指南 | https://docs.nvidia.com/ai-enterprise/planning-resource/licensing-guide/latest/licensing.html |
| **Run:ai 官方文件入口** | https://run-ai-docs.nvidia.com/ |
| Self-hosted 概覽 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/overview |
| Self-hosted 安裝總覽 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation |
| Support Matrix | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix |
| 叢集系統需求 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/system-requirements |
| Control Plane 系統需求 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/cp-system-requirements |
| 安裝前準備 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/preparations |
| Control Plane 安裝 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/install-control-plane |
| 叢集安裝（Helm） | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/helm-install |
| 網路需求 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/network-requirements |
| NVIDIA GPU Operator | https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html |
| What's New v2.25 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25 |
| GPU Operator Lifecycle | https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html |
