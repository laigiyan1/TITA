# NVAIE 文件庫索引

整理日期：2026-05-20  
目標：理解 NVIDIA AI Enterprise 架構，建置 VMware vSphere GPU 虛擬化平台  
GPU：H100 / H200（Hopper 架構）｜授權：已購買 NVAIE 訂閱

---

## 文件清單（doc-NVAIE/）

| 文件 | 說明 |
|------|------|
| [00-NVAIE-Overview-and-Architecture.md](00-NVAIE-Overview-and-Architecture.md) | NVAIE 平台概覽、兩層架構、支援矩陣 |
| [01-NVAIE-VMware-Prerequisites.md](01-NVAIE-VMware-Prerequisites.md) | 硬體/軟體先決條件、BIOS 設定 |
| [02-NVAIE-ESXi-VIB-Installation.md](02-NVAIE-ESXi-VIB-Installation.md) | Host Software VIB 安裝、Graphics Type 設定 |
| [03-NVAIE-License-System-DLS.md](03-NVAIE-License-System-DLS.md) | DLS/CLS 授權系統設定、Client Token 配置 |
| [04-NVAIE-First-VM-Creation.md](04-NVAIE-First-VM-Creation.md) | 建立 VM、Passthrough vs vGPU、Driver 安裝 |
| [05-NVAIE-Docker-Container-Toolkit.md](05-NVAIE-Docker-Container-Toolkit.md) | Docker Engine + NVIDIA Container Toolkit 安裝 |
| [06-NVAIE-Advanced-GPU-Config.md](06-NVAIE-Advanced-GPU-Config.md) | MIG、vGPU 排程、多 GPU、ECC 設定 |
| [07-NVAIE-Deployment-Checklist.md](07-NVAIE-Deployment-Checklist.md) | **完整部署檢查清單**（Passthrough + vGPU 兩種情境） |
| [08-NVAIE-GPU-Monitoring.md](08-NVAIE-GPU-Monitoring.md) | **GPU 監控**：Passthrough → DCGM + Prometheus + Grafana；vGPU → nvidia-smi 限制說明 |
| [NVIDIA-Enterprise-Reference-Architecture-May2025.pdf](NVIDIA-Enterprise-Reference-Architecture-May2025.pdf) | NVIDIA 官方企業參考架構白皮書（2025/05） |

---

## 架構圖清單（pic-NVAIE/）

| 檔案 | 說明 | 來源 |
|------|------|------|
| overview-no-icons.png | NVAIE 平台兩層架構圖 | Software Reference Architecture |
| software-stack-01.png | NVAIE 軟體堆疊圖（含 Kubernetes、GPU Operator） | Software Reference Architecture |
| ai-enterprise-factory-01.png | AI Factory 整體生態系架構圖 | AI Factory Design Guide |
| vgpu-arch.png | vGPU 架構圖（時間切片 + MIG 模式） | NVAIE vGPU Overview |
| vmware-001.png | vGPU 授權狀態驗證截圖 | vGPU VMware Deployment |
| vmware-002.png | NVIDIA Control Panel 授權介面截圖 | vGPU VMware Deployment |
| vmware-003.png | nvidia-smi 驗證截圖 | vGPU VMware Deployment |

---

## 兩個建置目標對應文件

### 目標 1：VMware VM + Direct Path I/O GPU Passthrough

閱讀順序：
1. `00` → 了解架構
2. `01` → 確認先決條件
3. `04` §3 → Passthrough 設定
4. `04` §5-6 → Ubuntu + Data Center Driver
5. `05` → Docker + Container Toolkit
6. `07` 情境一 → 按清單逐步執行
7. `08` → GPU 監控堆疊建置

---

### 目標 2：VMware VM + vGPU（含 License Server）

閱讀順序：
1. `00` → 了解架構
2. `01` → 確認先決條件
3. `02` → 安裝 VIB + 設定 Shared Direct
4. `03` → 部署 DLS License Server
5. `04` §4, §7-8 → vGPU VM 建立 + Grid Driver + Token
6. `05` → Docker + Container Toolkit
7. `07` 情境二 → 按清單逐步執行
8. `08` §4 → vGPU 監控限制說明（**DCGM 不支援 vGPU**；使用 nvidia-smi guest / hypervisor 視角）

---

## 關鍵決策：選擇 NVAIE 版本與 vSphere 版本

### NVAIE Infra 版本選擇（截至 2026-05-20）

| NVAIE 版本 | Driver | vGPU | 狀態 | 支援至 |
|------------|--------|------|------|--------|
| **Infra 8.1** | 595.71.05 | 20.1 | ✅ Active（Feature） | 2027/04 |
| **Infra 7.5 LTSB** | 580.159.03 | 19.5 | ✅ Active（LTSB） | 2028/07 |
| Infra 7.2 | — | — | ❌ Archived | — |
| Infra 6.x | — | — | ❌ EOL | — |

**建議：** 新建置優先評估 **Infra 8.1**（最新功能）或 **Infra 7.5 LTSB**（最長支援期）。7.2 已 archived，不應作為新建置基準。

### vSphere 版本支援

NVIDIA Support Matrix 未細分 vSphere 8.x U1/U2，8.1 與 7.5 對 vSphere 9.0 的描述措辭不同：

| vSphere 版本 | NVAIE Infra 8.1 | NVAIE Infra 7.5 LTSB |
|-------------|-----------------|----------------------|
| 8.0 and later | ✅ 支援 | ✅ 支援 |
| 9.0 and later | ✅ 支援（`9.0 and later`） | — |
| 9.0（指定版本） | — | ✅ 支援（`9.0`） |
| 7.x | ❌ 不支援 | ❌ 不支援 |

> 8.1 對 vSphere 9.0 的官方寫法為 `9.0 and later`；7.5 的寫法為 `9.0`。兩者均支援 vSphere 9.0，但表述不同，部署前請查閱所選版本的 Support Matrix 原文確認。

---

## 主要參考 URL

| 資源 | URL |
|------|-----|
| NVAIE 主頁 | https://docs.nvidia.com/ai-enterprise/index.html |
| VMware 部署指南 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/index.html |
| 相容矩陣 8.1 | https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html |
| 相容矩陣 7.5 LTSB | https://docs.nvidia.com/ai-enterprise/release-7/7.5/support/support-matrix.html |
| License System | https://docs.nvidia.com/license-system/latest/ |
| Container Toolkit | https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/ |
| NGC 登入 | https://ngc.nvidia.com |
| Licensing Portal | https://licensing.nvidia.com |
