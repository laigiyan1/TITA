# NVAIE VMware 部署先決條件

> 來源：https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/prereqs.html  
> 整理日期：2026-05-20

---

## 1. 硬體需求

### 1.1 伺服器需求
- **必要**：至少一張 NVIDIA Data Center GPU
- 伺服器必須是 **NVIDIA AI Enterprise Compatible NVIDIA-Certified System**
- 查詢認證伺服器：https://www.nvidia.com/en-us/data-center/products/certified-systems/

### 1.2 推薦 GPU 配置

| 使用案例 | 推薦 GPU |
|----------|----------|
| 主流伺服器 (1-8 GPU) | L4, L40S, H100 NVL, H200 NVL |
| 大型模型推論 (高容量) | 2x H200 或 Blackwell GPU |
| 大型模型訓練/推論 (HGX) | 4x 或 8x H200，或 8x Blackwell |

### 1.3 A100 GPU 特殊需求
- SR-IOV (Single Root I/O Virtualization)：**必須啟用**
- VT-d/IOMMU：**必須啟用**

---

## 2. 軟體需求

| 軟體 | 說明 |
|------|------|
| NVIDIA AI Enterprise License | 有效訂閱授權 |
| VMware ESXi Hypervisor ISO | 從 Broadcom/VMware 取得 |
| Ubuntu Server 22.04 / 24.04 LTS amd64 ISO | Guest OS（兩者均受 NVAIE 8.1 / 7.5 LTSB 支援；來源：[Support Matrix 8.1](https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html)、[Support Matrix 7.5 LTSB](https://docs.nvidia.com/ai-enterprise/release-7/7.5/support/support-matrix.html)） |
| NVIDIA AI Enterprise Host Software (VIB) | 從 NGC 下載 |
| NVIDIA vGPU Guest Driver | 從 NGC 下載（vGPU 模式） |

---

## 3. 伺服器 BIOS/設定建議

| 設定項目 | 建議值 | 說明 |
|----------|--------|------|
| Hyperthreading | 啟用 | 提升 vCPU 數量 |
| Power Setting | High Performance | 避免 CPU throttling |
| CPU Performance | Enterprise 或 High Throughput | |
| Memory Mapped I/O above 4GB | 啟用（建議）| 若 GPU 偵測有問題時必須開啟 |

---

## 4. 網路規劃注意事項

- ESXi Management Network：獨立 VMkernel NIC
- VM 資料網路：建議使用高速 NIC（10GbE+）
- 如使用 DLS 授權伺服器：VM 必須能連到 License Server（TCP 443/80）

---

## 5. VMware vSphere 版本支援

| NVAIE 版本 | 支援的 vSphere 版本（官方 Support Matrix 原文） | 狀態 |
|------------|------------------------------------------------|------|
| NVAIE Infra 8.1 | `8.0 and later, 9.0 and later` | Active（建議新建置） |
| NVAIE Infra 7.5 LTSB | `8.0 and later, 9.0` | Active LTSB（長期支援至 2028/07） |
| NVAIE Infra 7.2 | — | **已 Archived，不建議新建置** |
| NVAIE Infra 6.x | — | **已 EOL（2026/03）** |

> **注意：**
> - NVIDIA Support Matrix 未細分 vSphere 8.x U1/U2 更新版本，使用最新 vSphere 8.x 以確保相容性。
> - 8.1 的 vSphere 9.0 支援為 `9.0 and later`（不限定特定 update），7.5 的為 `9.0`（依 Support Matrix 原文）。
> - vSphere 7.x 不在 NVAIE 7.x/8.x 支援範圍內。

---

## 6. 查詢相容矩陣

官方相容矩陣（可依 GPU、Hypervisor、OS 篩選）：

- Infra 8.1（Feature Branch）：https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html  
- Infra 7.5 LTSB：https://docs.nvidia.com/ai-enterprise/release-7/7.5/support/support-matrix.html

---

## 7. 取得軟體的方式

1. 登入 **NVIDIA NGC**：https://ngc.nvidia.com
2. 使用企業帳號 API Key 存取 NVAIE 授權軟體
3. 下載項目：
   - `NVIDIA-AIE_ESXi_<version>.vib`（Host Software）
   - vGPU Guest Driver for Linux
   - NVIDIA License System (DLS) 映像檔

---

## 8. 授權入口

- **NVIDIA Licensing Portal**：取得並管理授權  
  https://licensing.nvidia.com
- **NVIDIA NGC**：下載軟體與容器映像  
  https://ngc.nvidia.com
- **Enterprise Support Portal**：技術支援  
  https://enterprise.nvidia.com
