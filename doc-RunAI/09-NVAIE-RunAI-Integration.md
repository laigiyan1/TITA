# NVAIE 與 Run:ai 整合架構說明

> 來源：https://docs.nvidia.com/ai-enterprise/latest/product-overview/index.html  
> 來源：https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/overview  
> 整理日期：2026-05-23  
> 目標版本：NVAIE Infrastructure 8.1 + Run:ai v2.25

---

## 1. Run:ai 在 NVAIE 中的定位

NVIDIA Run:ai 是 NVAIE **Infrastructure Layer（基礎架構層）** 的核心元件之一，負責：

- **AI 工作負載排程**：以 AI-aware 規則分配 GPU 資源給訓練、推論、互動式任務
- **GPU 資源管理**：跨多叢集統一管理 GPU 配額、佇列優先序與公平共享策略
- **工作負載監控**：提供 Web UI / API / CLI 管理介面，追蹤 GPU 使用率與工作進度

```
NVAIE 完整堆疊（以本案 Bare Metal 部署為例）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌───────────────────────────────────────────┐
│        Application Development Layer      │
│  NIM │ NeMo │ PyTorch / TF │ NGC Models  │  ← AI 開發工具層
└─────────────────────────────────────────-─┘
┌───────────────────────────────────────────┐
│      Infrastructure Management Layer      │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │         NVIDIA Run:ai v2.25         │  │  ← 工作負載排程
│  │  Control Plane ←→ Cluster Agent     │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  GPU Operator 26.3.1                      │  ← GPU 驅動自動化
│  Network Operator 26.1.1                  │  ← 高速網路管理
│  Container Toolkit 1.19.0                 │  ← 容器 GPU 存取
│  GPU Driver 595.71.05（R595）             │  ← 硬體驅動層
└───────────────────────────────────────────┘
┌───────────────────────────────────────────┐
│  Kubernetes（RKE2 / kubeadm）             │  ← 容器編排平台
└───────────────────────────────────────────┘
┌───────────────────────────────────────────┐
│  Ubuntu 22.04 LTS + 硬體（XE9780 / R670） │  ← 底層硬體與 OS
└───────────────────────────────────────────┘
```

> ⚠️ **Kubernetes 發行版說明**：上圖 RKE2 / kubeadm 為本案部署選型，均列於 Run:ai 官方支援矩陣。Dell 官方 2-8-9-400 brief（XE9680 參考平台）所列的 orchestration 路徑為 Upstream Kubernetes 或 Red Hat OpenShift Container Platform；若需對齊 Dell AI Factory endorsed stack，請由 Dell / NVIDIA SE 確認。

---

## 2. Run:ai 系統架構（詳細）

Run:ai 本身由兩個主要元件構成，均部署於 Kubernetes 之上：

```
┌──────────────────────────────────────────────────────┐
│              Run:ai Control Plane                    │
│  ─────────────────────────────────────────────────  │
│  • Web UI（工作負載管理、GPU 使用儀表板）               │
│  • REST API / CLI（runai）                           │
│  • Keycloak（SSO / 身份驗證）                        │
│  • PostgreSQL v16+（元數據儲存）                      │
│  • 內建 Analytics / Overview 儀表板（v2.25 新介面）   │
│  • 多叢集管理：單一 Control Plane 可管多個 Cluster    │
└──────────────────────┬───────────────────────────────┘
                       │
                       │  HTTPS（TLS 加密）
                       │  Cluster 元件 → Control Plane FQDN（outbound sync / metrics）
                       │  使用者 / CLI / UI → Control Plane（inbound HTTPS，需 Ingress）
                       │  傳輸：元數據 + GPU 指標（不含用戶資料集、模型、程式碼）
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│              Run:ai Cluster Agent                    │
│  ─────────────────────────────────────────────────  │
│  • Run:ai Scheduler（AI-aware GPU 排程引擎）          │
│  • Run:ai Agent（與 Control Plane 同步）             │
│  • 以 Kubernetes Operator 形式部署於 GPU 叢集         │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│   Kubernetes Worker Nodes（GPU 節點：XE9780）        │
│  ─────────────────────────────────────────────────  │
│  • NVIDIA GPU Operator（自動管理 Driver / Plugin）   │
│  • Prometheus（指標採集）                            │
│  • Ingress Controller、MetalLB                      │
└──────────────────────────────────────────────────────┘
```

---

## 3. 部署模式比較

Run:ai 支援兩種部署模式，均包含於 NVAIE 8.1 授權中：

| 比較項目 | Self-hosted（本地部署）⭐ 本案 | SaaS（NVIDIA 雲端管理） |
|----------|--------------------------|----------------------|
| **Control Plane 位置** | 客戶 Kubernetes 叢集（本地） | NVIDIA 雲端（客戶無需管理） |
| **Cluster Agent 位置** | 客戶 Kubernetes 叢集 | 客戶 Kubernetes 叢集 |
| **網路需求** | Control Plane 與 Cluster 在同一叢集可內網通訊 | Cluster 需要出站連線至 NVIDIA 雲端 |
| **版本管理** | 客戶自行升級 | NVIDIA 管理，自動更新 |
| **資料主權** | 完整本地控制 | 元數據傳至 NVIDIA 雲端 |
| **適用場景** | 高安全性、Airgapped、本地合規要求 | 快速部署、無維運負擔 |
| **NVAIE 授權涵蓋** | ✅ 自 NVAIE 首版起包含 | ✅ 自 NVAIE 8.1（2026-05）起正式包含 |

### 3.1 Self-hosted 架構（本案採用）

```
客戶資料中心（Dell 硬體）
┌─────────────────────────────────────────────────────┐
│  Kubernetes 叢集（RKE2 / kubeadm）                   │
│                                                     │
│  ┌──────────────────┐    ┌──────────────────────┐  │
│  │   Control Plane  │    │   Run:ai Cluster     │  │
│  │  （runai-backend  │◄──►│   （runai 命名空間）   │  │
│  │   命名空間）       │    │                      │  │
│  └──────────────────┘    └──────────────────────┘  │
│                                                     │
│  管理節點：Dell R670 ×2（K8s Master + Control Plane）│
│  GPU 節點：Dell XE9780 ×1（K8s Worker + Cluster）   │
└─────────────────────────────────────────────────────┘
```

### 3.2 SaaS 架構

```
NVIDIA 雲端
┌──────────────────────────────────┐
│  Run:ai Control Plane（NVIDIA 管）│
└──────────────────┬───────────────┘
                   │ Outbound HTTPS
                   ▼
客戶資料中心（Kubernetes 叢集）
┌──────────────────────────────────┐
│  Run:ai Cluster Agent            │
│  GPU Worker Nodes                │
└──────────────────────────────────┘
```

---

## 4. NVAIE 授權與 Run:ai

### 4.1 授權包含內容

**NVAIE 訂閱包含 Run:ai 的使用授權**，無需額外購買：

| 部署模式 | 包含於 NVAIE 版本 |
|----------|-----------------|
| Run:ai Self-hosted | 自 NVAIE 初版起包含 |
| Run:ai SaaS | **自 NVAIE Infrastructure 8.1（2026-05）起正式包含** |

兩種模式均受相同的 **NVAIE Enterprise SLA** 保障。

### 4.2 授權取得流程（Self-hosted）

```
1. 購買 NVAIE 訂閱（或使用 H100/H200 附贈授權）
       │
       ▼
2. 在 NVIDIA Enterprise Account 登錄
   → https://ngc.nvidia.com/（使用授權憑證上的 email 註冊）
       │
       ▼
3. NGC API Key 取得
   → 用於拉取 Run:ai Helm Chart 與容器映像
       │
       ▼
4. 部署 Run:ai（Helm 安裝）
   → Control Plane → Cluster Agent → 設定 License
       │
       ▼
5. 驗證授權啟用
   → Run:ai Web UI 確認 License 狀態
```

### 4.3 Run:ai 授權說明

> ⚠️ **以下授權細節（License 類型、Airgapped 啟用、自動驗證方式）因 NVAIE 合約版本、entitlement 類型與 support portal 設定而異，公開文件未完整佐證。請以 NVIDIA entitlement 憑證、Sales/SE 確認或 [NVIDIA Support Portal](https://enterprise.nvidia.com/) 為準。**

Run:ai 授權由 NVAIE 訂閱涵蓋，一般已知原則如下（**待與 NVIDIA SE 確認細節**）：
- 授權通常與叢集管理的 GPU 數量相關
- 需透過 NGC API Key 或 NVIDIA entitlement 啟用
- Airgapped 環境的授權啟用方式**需另行向 NVIDIA 確認**，公開文件未明確說明流程

---

## 5. Run:ai 核心功能與 GPU 管理機制

### 5.1 排程策略

| 功能 | 說明 |
|------|------|
| **Gang Scheduling** | 多 Pod 分散式訓練同時取得 GPU，避免死鎖等待 |
| **Fractional GPU** | 多工作負載共享單顆 GPU（依記憶體切割），提升利用率 |
| **Dynamic GPU Fraction** | 動態調整 GPU 記憶體分配比例 |
| **Preemption（佔先）** | 高優先工作到達時可暫停低優先工作，釋出 GPU |
| **Fair Share** | 多 Project / Department 間 GPU 公平分配策略 |
| **Over-Quota** | Project 在整體叢集空閒時可暫時超過配額使用 |

### 5.2 工作負載類型

| 類型 | 說明 | 典型使用 |
|------|------|---------|
| **Training（訓練）** | 批次型 GPU 任務，完成後結束 | 模型訓練、Fine-tuning |
| **Inference（推論）** | 常駐型服務，持續接受請求 | 生產推論服務 |
| **Interactive（互動）** | Jupyter Notebook / VS Code 互動開發環境 | 研究探索、Debug |
| **Distributed（分散式）** | 多節點多 GPU 訓練（支援 MPI、PyTorch DDP） | 大型模型訓練 |

### 5.3 與 NVAIE 其他元件的協作

```
Run:ai Scheduler
       │
       ├─ 透過 GPU Operator → 管理 GPU Driver / Device Plugin
       │
       ├─ 透過 Container Toolkit → 讓訓練容器存取 GPU
       │
       ├─ 整合 Prometheus → 匯出 GPU / 工作負載指標
       │
       └─ 整合 NIM / NeMo → 排程推論微服務與訓練任務
```

---

## 6. 多叢集架構（Multi-Cluster）

單一 Run:ai Control Plane 可管理多個 Cluster，適合：
- 跨資料中心 GPU 資源統一調度
- 開發叢集 / 生產叢集分離但統一監控
- 混合雲場景（本地 + 雲端 GPU 資源）

```
Run:ai Control Plane（單一管理入口）
          │
          ├── Cluster A（資料中心 A，H100 × 8）
          ├── Cluster B（資料中心 B，H200 × 4）
          └── Cluster C（雲端 AWS 節點）
```

> **本案規模**：單一叢集，Control Plane 與 Cluster Agent 共存於同一 K8s 叢集。

---

## 7. Run:ai v2.25 重要變更（NVAIE 8.1 對應版本）

來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25

| 變更項目 | 說明 |
|----------|------|
| **Legacy Grafana 移除** | Overview、Analytics、Consumption、Multi-cluster 等舊版 Grafana 儀表板已移除 |
| **新版內建儀表板** | 改用 Run:ai 原生儀表板，功能等效且整合更佳 |
| **SaaS 納入 NVAIE 授權** | Run:ai SaaS 正式成為 NVAIE 8.1 授權的一部分 |

---

## 8. 關鍵限制（Run:ai in NVAIE 場景）

| 限制 | 說明 |
|------|------|
| **僅支援資料中心 GPU（Passthrough）** | 不支援 vGPU 模式；不支援 DGX Spark、Jetson、工作站 GPU |
| **Kubernetes 必要** | Control Plane 與 Cluster Agent 均需 Kubernetes，不支援裸機直接部署 |
| **GPU Operator 必要** | Run:ai 依賴 GPU Operator 管理 GPU 資源，必須先安裝 |
| **NVAIE 授權必要** | 需要有效的 NVAIE 訂閱及 NGC API Key 才能拉取映像檔 |

---

## 參考來源

| 文件 | URL |
|------|-----|
| NVAIE 8.1 Release Notes | https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html |
| NVAIE Product Overview | https://docs.nvidia.com/ai-enterprise/latest/product-overview/index.html |
| Run:ai Self-hosted 概覽 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/overview |
| Run:ai What's New v2.25 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25 |
| Run:ai 安裝總覽 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation |
| NVAIE Infrastructure 生命週期 | https://docs.nvidia.com/ai-enterprise/lifecycle/latest/infrastructure-software.html |
| GPU Operator（NVAIE 模式） | https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/install-gpu-operator-nvaie.html |
