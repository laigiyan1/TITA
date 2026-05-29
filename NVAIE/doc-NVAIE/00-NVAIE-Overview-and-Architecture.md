# NVIDIA AI Enterprise (NVAIE) 概覽與架構

> 資料來源：NVIDIA 官方文件 docs.nvidia.com  
> 整理日期：2026-05-20  
> 目標環境：H100/H200 GPU、VMware vSphere、Docker on Ubuntu VM

---

## 1. NVAIE 是什麼？

NVIDIA AI Enterprise (NVAIE) 是一套雲原生 AI 軟體平台，提供：
- 最佳化效能的工具、函式庫與框架
- 企業級安全性與穩定性
- 支援從原型到生產環境的完整 AI 部署

官方文件入口：https://docs.nvidia.com/ai-enterprise/index.html

---

## 2. 兩層架構模型

### 2.1 Application Layer（應用層）
提供 AI 應用開發與部署能力：

| 組件 | 說明 |
|------|------|
| NVIDIA NIM Microservices | 加速 AI 模型推論的微服務 |
| NVIDIA NeMo | 大型語言模型訓練/微調框架 |
| AI Frameworks | TensorFlow、PyTorch、Triton 等最佳化版本 |
| Domain SDKs | 垂直領域 SDK（視訊、語音、醫療等） |
| Pre-trained Models | 透過 NGC 提供 100+ 預訓練模型 |
| NVIDIA Omniverse | 基於 OpenUSD 的 3D/數位孿生應用 |

### 2.2 Infrastructure Layer（基礎設施層）
處理底層 GPU、網路、Kubernetes 管理：

| 組件 | 說明 |
|------|------|
| Data Center GPU Driver | 實體/虛擬環境的 GPU 驅動 |
| vGPU Manager (VIB) | ESXi Host 上的 vGPU 管理器 |
| NVIDIA Container Toolkit | 讓 Docker 容器使用 GPU |
| GPU Operator | Kubernetes 的 GPU 生命週期管理 |
| Network Operator | 高吞吐量低延遲網路管理 |
| NVIDIA Run:ai | GPU 工作負載排程（自架或 SaaS） |
| Base Command Manager | 多節點叢集管理 |

> **架構圖：** 參見 `../pic-NVAIE/overview-no-icons.png`  
> **軟體堆疊圖：** 參見 `../pic-NVAIE/software-stack-01.png`

---

## 3. 兩層分離設計的優點

- **Infrastructure Layer 有版本號**，基礎更新不影響應用層開發
- 應用層與基礎設施層可獨立升級
- 彈性支援 Bare Metal、虛擬化、公有雲等多種部署模式

---

## 4. 支援的部署環境（NVAIE 8.1 / 7.5 LTSB）

| 部署類型 | 支援 |
|----------|------|
| Bare Metal (Ubuntu) | ✅ |
| VMware vSphere 8.0 and later | ✅ |
| VMware vSphere 9.0 and later（Infra 8.1）/ 9.0（Infra 7.5） | ✅ |
| VMware vSphere 7.x | ❌（NVAIE 7.x/8.x 不支援） |
| Public Cloud (AWS/GCP/Azure/OCI) | ✅ |
| Red Hat OpenShift | ✅ |
| Kubernetes (upstream) | ✅ |

---

## 5. 支援的 GPU（以 NVAIE 7.5 LTSB 為基準；Infra 8.1 可能支援額外型號，請查 8.1 Support Matrix）

| 架構 | 型號 |
|------|------|
| Hopper | **H100, H100 NVL, H200 NVL, H800, H800 NVL, H20** |
| Blackwell | RTX PRO 6000 Blackwell Server Edition |
| Ada Lovelace | L4, L40, L40S, L20, L2 |
| Ampere | A100, A800, A40, A30, A16, A10 |
| Turing/Volta | T4, V100 |

> **H100/H200 HGX 平台多 GPU VM 說明：**  
> 依據 NVIDIA Support Matrix (7.5/8.1)，在 VMware vSphere 上，HGX 平台的 vGPU for Compute 支援 **1、2、4、8 GPU per VM** 的配置。非 VMware vSphere 情境須依各 hypervisor / 平台的支援矩陣個別確認——例如 NVAIE 7.5 Support Matrix 明確指出「DGX platforms、HGX platforms with KVM hypervisors、IGX Orin」不支援 vGPU for Compute；實際可用組合仍需依部署的 NVAIE 版本與 Support Matrix 確認。

---

## 6. 兩種 GPU 虛擬化模式

### 6.1 Direct Path I/O（GPU Passthrough）
- 將 GPU 直接透傳給 VM，VM 看到完整物理 GPU
- 使用 Data Center Driver（非 vGPU Grid Driver）
- **不需要 vGPU License Server（NLS client leasing）**，但 NVAIE 軟體本身仍屬 per-GPU 訂閱授權——每個執行 NVAIE 軟體的 GPU 都必須持有有效的 NVAIE 訂閱。
- 性能最接近 Bare Metal

### 6.2 vGPU（Virtual GPU）
- 一張物理 GPU 分配給多個 VM
- 需要 NVIDIA AI Enterprise Host Software VIB 安裝在 ESXi
- **需要 NVAIE License Server**（CLS 或 DLS）
- VM 內使用 vGPU Guest Driver（安裝程式帶 `-grid.run` 後綴，故亦稱 Grid Driver）
- 支援 Live Migration（vMotion）

---

## 7. 授權模型

- **每 GPU 授權**：每台伺服器上安裝的每張 GPU 都需要一個授權
- **H100 PCIe/NVL**：購買時含 5 年訂閱授權
- **H200 NVL**：購買時含 5 年訂閱授權
- 授權種類：訂閱型（可更新）、永久型（含 5 年支援）

授權服務：
- **CLS (Cloud License Service)**：NVIDIA 雲端託管
- **DLS (Delegated License Service)**：客戶自建地端授權伺服器

---

## 8. 軟體版本（Active Releases，截至 2026-05-20）

### 8.1 NVAIE Infra 8.1（Feature Branch，建議新建置優先評估）

| 組件 | 版本 |
|------|------|
| GPU Driver | 595.71.05（R595 branch） |
| vGPU Software | 20.1 |
| 支援期限 | 2027 年 4 月 |
| Ubuntu 支援版本 | 20.04 LTS / 22.04 LTS / 24.04 LTS |

### 8.2 NVAIE Infra 7.5（Long-Term Support Branch，LTSB）

| 組件 | 版本 |
|------|------|
| GPU Driver | 580.159.03（R580 branch） |
| vGPU Software | 19.5 |
| 支援期限 | 2028 年 7 月 |
| Ubuntu 支援版本 | 20.04 LTS / 22.04 LTS / 24.04 LTS |

> **注意：** NVAIE 7.2 已 archived，不應作為新建置基準。請選擇 Infra 8.1（最新 Feature Branch）或 Infra 7.5 LTSB（3 年長期支援）。
> 
> - 若優先考量長期穩定性與最長支援週期 → 選 **7.5 LTSB**  
> - 若希望使用最新功能與最新 Driver → 選 **8.1**

---

## 9. 參考連結

| 文件 | URL |
|------|-----|
| NVAIE 主頁 | https://docs.nvidia.com/ai-enterprise/index.html |
| Platform Overview | https://docs.nvidia.com/ai-enterprise/reference-architecture/latest/platform-overview.html |
| Software Reference Architecture | https://docs.nvidia.com/ai-enterprise/reference-architecture/latest/index.html |
| Infrastructure Support Matrix 8.1 | https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html |
| Infrastructure Support Matrix 7.5 (LTSB) | https://docs.nvidia.com/ai-enterprise/release-7/7.5/support/support-matrix.html |
| AI Factory White Paper | https://docs.nvidia.com/ai-enterprise/planning-resource/ai-factory-white-paper/latest/ai-factory-overview.html |
| Enterprise Reference Architectures | https://docs.nvidia.com/enterprise-reference-architectures/index.html |
