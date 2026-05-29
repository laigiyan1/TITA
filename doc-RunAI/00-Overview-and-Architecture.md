# NVIDIA Run:ai 產品概覽與架構

> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/overview  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation  
> 整理日期：2026-05-20  
> 目標版本：NVIDIA Run:ai v2.25

---

## 1. 產品定位

NVIDIA Run:ai 是一套 **AI 操作平台**（AI Operations Platform），透過動態資源調度最大化 GPU 使用效率，為 AI 工作負載提供企業級的資源管理、排程與監控能力。

**核心價值：**
- AI 原生工作負載調度（AI-Native Workload Orchestration）
- 統一 AI 基礎架構管理（Unified AI Infrastructure Management）
- 跨多集群的集中式 GPU 資源管理
- 開放式架構（API-first）

---

## 2. 系統架構

Run:ai 由兩個主要元件組成，均部署於 Kubernetes 叢集之上：

```
┌─────────────────────────────────────────────────┐
│           NVIDIA Run:ai Control Plane            │
│  (資源管理 / 工作負載提交 / 監控儀表板 / 多集群管理)  │
│  ─ Web UI ─ REST API ─ CLI ─                    │
│  ─ PostgreSQL ─ Keycloak ─ 內建監控 / Analytics ─  │
└─────────────────┬───────────────────────────────┘
                  │ 加密 SSL 連線（Cluster → Control Plane）
                  │ 僅傳輸 Metadata 與 Metrics（不含用戶資料集）
                  ▼
┌─────────────────────────────────────────────────┐
│           NVIDIA Run:ai Cluster                  │
│  (排程引擎 / GPU 資源分配 / 工作負載執行管理)        │
│  ─ Run:ai Scheduler ─ Run:ai Agent ─            │
│  ─ 部署為 Kubernetes Operator ─                  │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│         Kubernetes 叢集（GPU Worker Nodes）        │
│  ─ NVIDIA GPU Operator ─ Prometheus ─           │
│  ─ Ingress Controller ─ MetalLB ─               │
└─────────────────────────────────────────────────┘
```

---

## 3. 核心元件說明

### 3.1 Run:ai Control Plane（控制平面）

| 職責 | 說明 |
|------|------|
| 資源管理 | 跨多集群的 GPU 資源配額與分配 |
| 工作負載提交 | 接收 Researcher 提交的 AI 工作負載 |
| 監控與分析 | 提供 GPU 使用率、工作負載狀態儀表板 |
| 多集群管理 | 單一控制平面可管理多個地點的 GPU 集群 |
| 存取介面 | Web UI、REST API、CLI（runai CLI） |

**內部元件：**
- **PostgreSQL v16+**：儲存元數據（可使用外部 PostgreSQL）
- **Keycloak**：身份驗證與 SSO 管理
- **監控 / 分析元件**：提供 Run:ai 內建監控與儀表板能力（Overview、Analytics 等頁面）  
  > ⚠️ **v2.25 變更**：legacy Grafana dashboards（Overview、Analytics、Consumption、Multi-cluster）已於 v2.25 移除，改用 Run:ai 新版內建儀表板介面。  
  > 來源：[What's New v2.25](https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25)

### 3.2 Run:ai Cluster（叢集代理）

| 職責 | 說明 |
|------|------|
| 工作負載排程 | 應用 AI-aware 規則，高效率分配 GPU 資源 |
| 工作負載管理 | 管理 Researcher 程式碼（以 K8s Container 執行）與系統資源 |
| 部署方式 | 以 Kubernetes Operator 形式安裝，自動化部署與設定 |
| 通訊方式 | 向外（Outbound-only）以加密 SSL 連線同步至 Control Plane |
| 資料安全 | **僅傳輸元數據與運作指標，不傳輸用戶資料集或模型** |

---

## 4. 部署模式

### 4.1 SaaS 模式（雲端託管）

```
NVIDIA 雲端 ←───── Run:ai Control Plane（NVIDIA 管理）
      │
      │ Outbound SSL
      ▼
客戶端 Kubernetes 叢集（Run:ai Cluster 部署於此）
```

- Control Plane 由 NVIDIA 管理，客戶僅需部署 Cluster 端
- 始終使用最新功能
- 需要出站網路連線至 NVIDIA 雲端

### 4.2 Self-hosted 模式（本地部署）⭐ 本案採用

```
客戶資料中心
┌──────────────────────────────────────────┐
│  Kubernetes 叢集                          │
│  ┌────────────────┐  ┌─────────────────┐ │
│  │ Control Plane  │  │   Run:ai Cluster│ │
│  │ (runai-backend │  │   (runai 命名空間)│ │
│  │  命名空間)      │  │                 │ │
│  └────────────────┘  └─────────────────┘ │
│                                          │
│  (Control Plane 與 Cluster 通常在同一個 K8s 叢集) │
└──────────────────────────────────────────┘
```

- Control Plane 與 Cluster **部署於同一 Kubernetes 叢集**（典型設定）
- 完全本地控制，適合高度安全性需求環境
- 版本管理由客戶負責

### 4.3 本案硬體對應

| 設備 | 作業系統 | K8s 角色 | Run:ai 角色 |
|------|----------|----------|-------------|
| Dell R670 #1 | Ubuntu 22.04 LTS 或 24.04 LTS | K8s Control Plane (Master) | Run:ai Control Plane |
| Dell R670 #2 | Ubuntu 22.04 LTS 或 24.04 LTS | K8s Control Plane (Master) / Worker | 高可用備援 |
| Dell XE9780 | **Ubuntu 24.04 LTS**（Dell 認證版本） | K8s Worker Node（GPU） | Run:ai Cluster（GPU 調度） |

> **注意**：典型小型環境中，Control Plane 與 Cluster 可共存於同一 K8s 叢集。R670 承載管理元件，XE9780 承載 GPU 工作負載。

---

## 5. 通訊流程

### 5.1 Cluster → Control Plane 通訊

- **方向**：Cluster 主動連線至 Control Plane（Outbound only）
- **協定**：HTTPS（TLS 加密）
- **內容**：
  - 工作負載狀態元數據同步
  - GPU 使用率指標
  - 節點健康狀態
- **注意**：**不傳輸用戶資料集、程式碼或模型權重**

### 5.2 用戶與 Control Plane 通訊

- Researcher 透過 CLI / API 提交工作負載至 Control Plane
- Control Plane 將工作負載傳送至對應 Cluster 的 K8s API Server
- 結果回傳至 Control Plane 儀表板

---

## 6. 多集群管理能力

單一 Run:ai Control Plane 可管理多個 Run:ai Cluster，適合：
- 多資料中心 GPU 集群統一管理
- 不同地點、不同網段的 GPU 節點集中調度
- 本案（小型環境）：僅一個 K8s 叢集，Control Plane + Cluster 共存

---

## 7. 關鍵限制

| 限制項目 | 說明 |
|----------|------|
| GPU 支援 | 僅支援資料中心 GPU（透過 Passthrough）；**不支援 vGPU** |
| 不支援硬體 | DGX Spark、NVIDIA Jetson、工作站 GPU |
| GPU Operator | **必須**安裝 NVIDIA GPU Operator（Run:ai 依賴此元件管理 GPU） |
| K8s 必要性 | Control Plane 與 Cluster 均**必須**部署於 Kubernetes |
| 授權 | 需要 NVIDIA NGC 帳號取得映像檔與 Helm Charts |

---

## 8. Dell AI Factory 方案框架

### 8.1 方案定位

本案 GPU 節點採用 **Dell PowerEdge XE9780**，定位為參考 NVIDIA Enterprise Reference Architecture（ERA）2-8-9-400 的 HGX 節點設計模式。「Dell AI Factory」是 Dell Technologies 與 NVIDIA 共同定義的企業 AI 基礎架構方案品牌，涵蓋硬體選型、網路設計、軟體堆疊與部署驗證標準。

> ⚠️ **Endorsement 說明**：NVIDIA ERA White Paper 說明的是通用 HGX 2-8-9-400 設計模式（非 Dell 專屬）；Dell 官方 2-8-9-400 brief 及 NVIDIA ERA index 中明確列出的 Dell 參考平台為 **Dell PowerEdge XE9680**，非本案的 XE9780。本案應描述為「**參考 NVIDIA ERA 2-8-9-400 HGX 設計模式**」，而非「已獲 Dell AI Factory ERA endorsement 覆蓋」。XE9780 是否屬於 Dell AI Factory 的正式 endorsed configuration，**需透過 Dell / NVIDIA SE、正式 BOM 或最新 Dell 官方文件確認**。

**2-8-9-400 命名規則**（描述單台 HGX GPU 伺服器的規格，非整個叢集）：

| 數字 | 含義 | ERA White Paper 規格 | 本案（Dell XE9780，依 BOM 確認） |
|------|------|---------------------|--------------------------------|
| **2** | 每台伺服器 CPU 數量 | 2× CPU | 2× CPU（依 BOM） |
| **8** | 每台伺服器 GPU 數量 | 8× GPU（HGX H100/H200/B200）| 8× GPU（B200 SXM6 / B300 NVL8，依 Ubuntu 認證佐證；最終以採購 BOM 確認）|
| **9** | 每台伺服器 NIC 數量 | 8× B3140H SuperNIC（E-W）+ 1× B3220 DPU（N-S）| 依 BOM 確認 |
| **400** | 每張東西向 NIC 頻寬（Gb/s） | 400Gb/s × 8 = **3.2Tb/s** | 依 BOM 確認 |

> 參照圖片：[era-2-8-9-400-single-server.png](../pic-RunAI/era-2-8-9-400-single-server.png)（NVIDIA ERA White Paper 通用 HGX 節點示意圖，來源：[Key Building Blocks](https://docs.nvidia.com/enterprise-reference-architectures/white-paper/latest/key-building-blocks.html)；非 XE9780 或 XE9680 專屬）

```
Dell AI Factory / NVIDIA ERA 參考架構對齊（本案對應）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌─────────────────────────────────────────────────┐
│  Application Layer（AI 應用開發層）              │
│  NIM Microservices │ NeMo │ PyTorch / TF         │
│  NGC 預訓練模型 │ Domain SDKs                    │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│  Orchestration Layer（工作負載調度層）            │
│  NVIDIA Run:ai v2.25  ← 本文件庫核心             │
│  Control Plane（on R670）+ Cluster（on XE9780）  │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│  Infrastructure Management Layer（基礎架構層）   │
│  NVAIE Infrastructure 8.1                        │
│  GPU Operator 26.3.1 │ Network Operator 26.1.1  │
│  Container Toolkit 1.19.0 │ GPU Driver R595      │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│  Platform Layer（容器平台層）                    │
│  Kubernetes（RKE2 / kubeadm）+ Ubuntu 24.04 LTS  │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│  Hardware Layer（本案硬體）                      │
│  Dell PowerEdge XE9780 × 1                       │
│    ← 參考 2-8-9-400 HGX 節點設計                │
│    ← BOM / Dell AI Factory endorsement 待確認   │
│    └─ 2× CPU                                     │
│    └─ 8× GPU（B200 SXM6 / B300 NVL8，依 Ubuntu 認證）│
│    └─ NIC 配置依 BOM 確認                        │
│    └─ NVSwitch 全互聯（節點內 GPU-to-GPU）        │
│  Dell PowerEdge R670 × 2（本案管理節點）          │
│    ← Dell 2-8-9-400 brief 參考配置為 4 台 R670   │
│    ← 本案為縮減部署（2 台）                       │
│    └─ K8s Control Plane + Run:ai Control Plane   │
└─────────────────────────────────────────────────┘
```

> **NVIDIA ERA 參考資料**：[ERA White Paper — Key Building Blocks](https://docs.nvidia.com/enterprise-reference-architectures/white-paper/latest/key-building-blocks.html)  
> **Dell 2-8-9-400 Brief（XE9680 參考平台）**：[Dell AI Factory 2-8-9-400 Brief](https://www.delltechnologies.com/asset/en-us/solutions/infrastructure-solutions/briefs-summaries/nvidia-2-8-9-400-configuration-era-endorsed-for-the-dell-ai-factory-with-nvidia-brief.pdf)

### 8.2 本案網路拓撲

```
本案網路分層（參考 NVIDIA ERA 三網分離設計）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. OOB 管理網路（Out-of-Band）
   ────────────────────────────────────────────────
   用途：iDRAC 遠端管理、硬體監控、韌體更新
   設備：R670 × 2 iDRAC Port ↔ XE9780 iDRAC Port
        接至管理交換器（1GbE）
   說明：與業務網路實體隔離

2. 資料/管理網路（In-Band / K8s 叢集網路）
   ────────────────────────────────────────────────
   用途：K8s API、Pod 網路（CNI）、Run:ai 控制流量
         使用者存取 Run:ai Web UI / API
   設備：R670 × 2 ↔ XE9780（GbE 或 10/25GbE）
   配置：MTU 9000 Jumbo Frame（K8s 建議）
         MetalLB IP Pool 規劃（LoadBalancer Service）

3. 高速互連網路（GPU East-West / High-Speed Fabric）
   ────────────────────────────────────────────────
   用途：GPU 分散式訓練節點間通訊（NCCL / RDMA）
   
   【NVIDIA ERA 2-8-9-400 NIC 設計模式（每台 HGX 節點）】
   東西向（GPU Compute E-W）：
     8× NVIDIA B3140H SuperNIC，每張 400Gb/s
     → 合計東西向頻寬：8 × 400Gb/s = 3.2Tb/s per server
     → 多節點時透過高速 Spine 交換器連接
       ‣ NVIDIA ERA White Paper generic topology：SN5600
       ‣ Dell 2-8-9-400 brief（XE9680）：Spectrum-4 SN5610
       ‣ 本案（XE9780）實際交換器型號：**依採購 BOM 確認**
   
   南北向（Storage / Management N-S）：
     1× NVIDIA B3220 DPU，2× 200Gb/s
     → 承載 Storage Traffic 及 User/Control 管理流量

   【單節點場景（本案當前）】
   XE9780 節點內：NVSwitch 全互聯（GPU-to-GPU）
   → 8 顆 GPU 透過 NVSwitch 直接通訊（900GB/s GPU-to-GPU）
   → 外部高速 NIC（若配備）仍建議連接上行交換器供未來擴充
   
   【多節點擴充時（未來）】
   XE9780 × N 節點共享高速 Spine 交換器
   → NCCL 跨節點 RDMA / GPUDirect RDMA 透過 NIC
   → OOB 管理網路：SN2201 或同等管理交換器
```

> 參照圖片：[era-scalable-unit-topology.png](../pic-RunAI/era-scalable-unit-topology.png)（**NVIDIA ERA White Paper 通用拓撲示意圖**：SN2201 OOB + SN5600 Consolidated；注意 Dell 2-8-9-400 brief 的實際網路設備為 Spectrum-4 SN5610，與本 White Paper generic topology 不同，本案實際型號以 BOM 為準）

---

## 參考來源

- [Self-hosted 概覽](https://run-ai-docs.nvidia.com/self-hosted/getting-started/overview)
- [安裝總覽](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation)
- [Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix)
- [NVIDIA ERA Index](https://docs.nvidia.com/enterprise-reference-architectures/index.html)
- [NVIDIA HGX AI Factory Reference Architecture](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/index.html)
