# NVIDIA AI Enterprise（NVAIE）平台概覽與架構

> 來源：https://docs.nvidia.com/ai-enterprise/latest/product-overview/index.html  
> 來源：https://docs.nvidia.com/ai-enterprise/software/latest/overview.html  
> 來源：https://docs.nvidia.com/ai-enterprise/planning-resource/licensing-guide/latest/licensing.html  
> 來源：https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html  
> 整理日期：2026-05-23  
> 目標版本：NVIDIA AI Enterprise Infrastructure 8.1（最新 Active 版本）

---

## 1. NVAIE 是什麼？

**NVIDIA AI Enterprise（NVAIE）** 是 NVIDIA 推出的端對端 AI 軟體訂閱平台，涵蓋從原型開發到生產部署的完整 AI 生命週期，適用於資料中心、雲端與邊緣環境。

NVAIE 的核心價值在於：
- 提供**企業級支援 SLA**（Service Level Agreement）
- 整合**完整軟體堆疊**（AI 框架、推論微服務、GPU 管理工具）
- 採用**獨立版本週期**的可組合式架構，讓應用層與基礎架構層可以分開升級
- 支援 **Bare Metal、VMware vSphere、主流雲端平台**（AWS、GCP、Azure、OCI）

---

## 2. 兩層可組合式架構（Composable Stack）

NVAIE 採用「兩層獨立版本」架構設計：

```
╔══════════════════════════════════════════════════════════════════════╗
║              NVIDIA AI Enterprise — Composable Stack                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │              Application Development Layer                    │   ║
║  │                                                              │   ║
║  │  NIM Microservices │ NeMo Framework │ Omniverse              │   ║
║  │  AI Frameworks (PyTorch / TensorFlow) │ Pre-trained Models   │   ║
║  │  Domain SDKs │ CUDA-X / CUDA Libraries                       │   ║
║  │                                                              │   ║
║  │  版本週期：~9 個月（Production Branch）或 ~36 個月（LTSB）       │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                           ▲  相互獨立，可分開升級                     ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │              Infrastructure Management Layer                 │   ║
║  │                                                              │   ║
║  │  GPU Drivers │ NVIDIA GPU Operator │ Network Operator        │   ║
║  │  Container Toolkit │ NVIDIA Run:ai │ Base Command Manager    │   ║
║  │  vGPU / MIG │ NIM Operator │ DPU Operator (DOCA)            │   ║
║  │                                                              │   ║
║  │  版本週期：~12 個月（Feature Branch）或 ~36 個月（LTSB）         │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  底層：NVIDIA Certified Systems（伺服器硬體）+ Kubernetes + OS        ║
╚══════════════════════════════════════════════════════════════════════╝
```

> 參考圖片：[nvaie-composable-stack.svg](../pic-RunAI/nvaie-composable-stack.svg)（官方 SVG）

**兩層獨立的意義：**
- 更新 NIM 微服務不需要同步升級 GPU Driver
- Infrastructure 層可先升級至新版 GPU Operator，不影響正在運行的 AI 框架
- 企業可依需求選擇 Feature Branch（最新功能）、Production Branch（生產穩定）或 LTSB（長期穩定）

---

## 3. Application Layer 元件說明

| 元件 | 說明 | 定位 |
|------|------|------|
| **NIM Microservices** | NVIDIA Inference Microservices，容器化 AI 推論服務（LLM、視覺、語音等） | 生產推論 |
| **NeMo Framework** | 大型語言模型訓練、微調、RAG 應用開發框架 | 模型開發 |
| **NVIDIA Omniverse** | 工業 / 物理 AI 仿真平台，支援 Digital Twin | 物理 AI |
| **AI Frameworks** | PyTorch、TensorFlow、JAX（NVIDIA 優化版本） | 模型訓練 |
| **Pre-trained Models** | NGC Catalog 中的預訓練模型（CV、NLP、語音等） | 快速啟動 |
| **Domain SDKs** | TAO Toolkit、Riva、Merlin 等垂直領域 SDK | 垂直應用 |
| **CUDA-X / CUDA** | 底層運算加速函式庫（cuDNN、cuBLAS、NCCL 等） | 基礎加速 |

---

## 4. Infrastructure Layer 元件說明

| 元件 | 說明 | 本案相關性 |
|------|------|-----------|
| **GPU Driver（R595）** | NVIDIA 資料中心 GPU 驅動程式，支援 Passthrough 與 vGPU 模式 | ✅ 必裝 |
| **NVIDIA GPU Operator** | Kubernetes Operator，自動化管理 GPU Driver、Container Toolkit、Device Plugin | ✅ 必裝 |
| **NVIDIA Network Operator** | 管理 InfiniBand / RoCE 高速網路（ConnectX 系列）的 Kubernetes Operator | 視需求 |
| **NVIDIA Container Toolkit** | 讓容器可存取 GPU 資源的底層工具（含 nvidia-container-runtime） | ✅ 必裝（GPU Operator 自動安裝） |
| **NVIDIA Run:ai** | AI 工作負載排程與 GPU 資源管理平台（見文件 09） | ✅ 本案核心 |
| **Base Command Manager** | 企業級 GPU 叢集生命週期管理工具（Provisioning、Monitoring） | 可選 |
| **vGPU / MIG** | GPU 虛擬化與分割技術 | 視部署模式 |
| **NIM Operator** | 自動化部署 NIM 微服務的 Kubernetes Operator | 推論場景 |
| **DPU Operator（DOCA）** | 管理 BlueField DPU 的 Kubernetes Operator | 視硬體 |

> 參考圖片：[nvaie-software-stack.png](../pic-RunAI/nvaie-software-stack.png)、[nvaie-platform-overview.png](../pic-RunAI/nvaie-platform-overview.png)

---

## 5. NVAIE 版本分支與版本矩陣

### 5.1 版本分支說明

NVAIE 的 Application Layer 與 Infrastructure Layer **各有獨立的版本分支定義與生命週期**，不可混用：

#### Application Layer 版本分支

來源：https://docs.nvidia.com/ai-enterprise/lifecycle/latest/application-software.html

| 分支類型 | 說明 | 支援週期（約） |
|----------|------|---------------|
| **Feature Branch（FB）** | 最新功能，快速迭代 | ~1 個月 |
| **Production Branch（PB）** | 穩定，適合生產環境 | ~9 個月 |
| **Long-Term Support Branch（LTSB）** | 長期穩定，適合高穩定性需求 | ~36 個月 |

#### Infrastructure Layer 版本分支

來源：https://docs.nvidia.com/ai-enterprise/lifecycle/latest/infrastructure-software.html

Infrastructure Layer 以「**Infrastructure Branch**」為單位（如 Infra 8.x、7.x、4.x），每個 Branch 包含若干 Major / Minor Release。官方 Lifecycle Explorer 中使用 FB / PB 作為**對齊分組標籤**（標示哪些 Infrastructure Branch 與哪些 Application Branch 相容），**並非 Infrastructure 自身的版本類型名稱**。

| Infrastructure Branch | GPU Driver 系列 | 支援類型 | 一般 EOL 週期 |
|-----------------------|----------------|----------|--------------|
| **Infra 8.x（R595）** | R595 | 一般發行版 | ~12 個月 |
| **Infra 7.x（R580）** | R580 | LTSB（長期穩定） | ~36 個月 |
| **Infra 4.x（R535）** | R535 | LTSB（長期穩定） | ~36 個月 |
| Infra 6.x（R570） | R570 | **EOL** | 已終止 |

> ⚠️ **注意**：請勿將 Application Layer 的 FB（~1 個月）/ PB（~9 個月）週期直接套用至 Infrastructure Layer。兩層獨立定義，詳見官方 [Lifecycle Policy](https://docs.nvidia.com/ai-enterprise/lifecycle/latest/infrastructure-software.html)。

> 參考圖片：[nvaie-infra-stack-alignment.svg](../pic-RunAI/nvaie-infra-stack-alignment.svg)  
> ⚠️ **圖中版本號為示例（example），用於說明版本分支的對齊模式，非實際版本矩陣。實際版本以 [NVAIE 8.1 Release Notes](https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html) 與 [Lifecycle Explorer](https://docs.nvidia.com/ai-enterprise/lifecycle/latest/infrastructure-software.html) 為準。**

### 5.2 目前 Active 版本（截至 2026-05）

| 版本 | 支援狀態 | GPU Driver | Run:ai 版本 | GPU Operator | EOL |
|------|---------|-----------|------------|-------------|-----|
| **Infra 8.1** ⭐ 最新 | 一般 Infrastructure Branch | 595.71.05 | 2.25 | 26.3.1 | 請查 Lifecycle Explorer |
| **Infra 7.5** | LTSB | 580.159.03 | 請查 Lifecycle Explorer | 請查 Lifecycle Explorer | 請查 Lifecycle Explorer |
| **Infra 4.10** | LTSB | 535.309.01 | 請查 Lifecycle Explorer | 請查 Lifecycle Explorer | 請查 Lifecycle Explorer |
| Infra 6.x | **EOL** | R570 | — | — | 已終止 |

> **Infra 8.1 的 GPU Driver、Run:ai、GPU Operator、Network Operator 等元件版本有 [NVAIE 8.1 Release Notes](https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html) 直接佐證；EOL 日期屬生命週期資訊，需另行透過 [Lifecycle Explorer](https://docs.nvidia.com/ai-enterprise/lifecycle/latest/infrastructure-software.html) 查證，不應以 Release Notes 為來源。**  
> **Infra 7.5 / 4.10 的 GPU Driver 版本可由 Lifecycle Explorer 頁面支持；Run:ai、GPU Operator 與 EOL 欄位則需透過 [Lifecycle Explorer](https://docs.nvidia.com/ai-enterprise/lifecycle/latest/infrastructure-software.html) 或 [NGC Infrastructure Collections](https://catalog.ngc.nvidia.com/) 逐項查證，上表未填入以避免誤導。**

### 5.3 NVAIE 8.1 完整元件版本清單

來源：https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html

| 元件 | 版本 | 備注 |
|------|------|------|
| Data Center GPU Driver | **595.71.05** | R595 Production Branch |
| vGPU Software | **20.1** | 含 Virtual GPU Manager + Guest Driver |
| NVIDIA Run:ai（Self-hosted） | **2.25** | 同時支援 Self-hosted 與 SaaS |
| NVIDIA Run:ai（SaaS） | **2.25** | 8.1 起正式納入 NVAIE 授權 |
| GPU Operator | **26.3.1** | Patch update |
| Network Operator | **26.1.1** | Patch update |
| DPU Operator（DPF） | **25.10.1** | — |
| NIM Operator | **3.1.0** | — |
| Container Toolkit | **1.19.0** | — |

---

## 6. NVAIE 授權模式

### 6.1 授權計費方式

| 授權類型 | 說明 | 適用場景 |
|----------|------|----------|
| **年訂閱（Annual Subscription）** | 每年按 GPU 數量訂閱 | 標準企業採購 |
| **雲端消費型（Consumption-based）** | 依使用量計費，透過 AWS/GCP/Azure Marketplace | 雲端部署 |
| **永久授權（Perpetual）** | 一次性購買 + 強制 5 年支援合約 | 長期投資 |

**授權計費單位：每顆安裝在伺服器上的 GPU**  
- 伺服器若無 GPU，則改為每台伺服器/實例一個訂閱

### 6.2 捆綁授權 GPU

以下 GPU 購買時即附贈 NVAIE 訂閱（無需額外購買）：

| GPU 型號 | 附贈 NVAIE 訂閱期限 |
|---------|-------------------|
| NVIDIA H100 | 5 年 |
| NVIDIA H200 | 5 年 |
| NVIDIA A800 | 3 年 |

> 訂閱起算日：GPU 出貨給 OEM 合作夥伴後 90 天

### 6.3 企業支援等級

| 支援等級 | 服務內容 | 包含於訂閱 |
|----------|----------|-----------|
| **Business Standard** | 本地工作時間工程師支援，4 小時首次回應，含所有更新與安全修補 | ✅ 預設包含 |
| **Business Critical** | 24/7 全天候支援，加速回應時間 | 需加購 |
| **Technical Account Manager（TAM）** | 專屬 NVIDIA 技術顧問，策略規劃與部署協助 | 需加購 |

---

## 7. 支援硬體

NVAIE 支援 **NVIDIA Certified Systems**（通過 NVIDIA 認證的伺服器）。

**NVAIE 8.0+ 起不再支援的 GPU（已改為僅 LTSB 7.5 / 4.10 支援）：**
- Tesla V100
- RTX 4000 SFF Ada
- RTX A4000
- Quadro RTX 系列

**本案硬體（Dell PowerEdge XE9780 + 資料中心 GPU）原則上應依實際 BOM 對照 NVAIE / GPU Operator 支援矩陣；依 Canonical Ubuntu 認證頁面（Cert ID: 202510-37982），XE9780 認證的 GPU 為 B200 SXM6 或 B300 NVL8，最終型號、driver branch 與 GPU Operator 版本需於採購與部署前以 BOM 確認。**

---

## 8. 部署環境支援

| 部署平台 | 支援狀態 |
|----------|---------|
| Bare Metal（裸機 Kubernetes） | ✅ 完整支援（本案採用） |
| VMware vSphere（vGPU 模式） | ✅ 支援 |
| VMware vSphere（Passthrough 模式） | ✅ 支援 |
| AWS / Google Cloud / Azure / OCI | ✅ 支援（VMI 模式） |
| Red Hat AI Factory | ✅ 整合支援 |

---

## 參考來源

### Dell AI Factory × NVIDIA ERA

| 文件 | URL |
|------|-----|
| **NVIDIA ERA Index（Enterprise Reference Architectures）** | https://docs.nvidia.com/enterprise-reference-architectures/index.html |
| **NVIDIA HGX AI Factory Reference Architecture** | https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/index.html |
| **Dell AI Factory 2-8-9-400 Brief（XE9680 參考平台）** | https://www.delltechnologies.com/asset/en-us/solutions/infrastructure-solutions/briefs-summaries/nvidia-2-8-9-400-configuration-era-endorsed-for-the-dell-ai-factory-with-nvidia-brief.pdf |

### NVAIE 官方文件

| 文件 | URL |
|------|-----|
| NVAIE 產品概覽 | https://docs.nvidia.com/ai-enterprise/latest/product-overview/index.html |
| NVAIE 軟體元件清單 | https://docs.nvidia.com/ai-enterprise/software/latest/overview.html |
| NVAIE 8.1 Release Notes | https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html |
| Infrastructure 版本生命週期 | https://docs.nvidia.com/ai-enterprise/lifecycle/latest/infrastructure-software.html |
| 授權指南 | https://docs.nvidia.com/ai-enterprise/planning-resource/licensing-guide/latest/licensing.html |
| Reference Architecture 入口 | https://docs.nvidia.com/ai-enterprise/reference-architecture/latest/introduction.html |
| GPU Operator（NVAIE 模式） | https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/install-gpu-operator-nvaie.html |
