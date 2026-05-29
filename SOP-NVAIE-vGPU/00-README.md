# SOP：VMware ESXi 8.0U3 + vCenter 8.0U3 上建置 NVIDIA vGPU（H200 NVL × 2）

> 撰寫日期：2026-05-28
> 目標：讓具備 VMware 基礎觀念的工程師可依本手冊，將兩張 NVIDIA H200 NVL 上的 vGPU 資源透過 profile 分派給 **2 個 Ubuntu 24.04 LTS 測試 VM**，並完成 vGPU 狀態驗證。
> 適用 NVAIE 版本：**Infra 8.1（建議，主要範例）** 或 **Infra 7.5 LTSB（長期支援替代）**
> 來源：以 `../doc-NVAIE/` 內 NVIDIA 官方資料整理為基礎，並對照 [NVAIE VMware 部署文件](https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/index.html) 與 [Support Matrix 8.1](https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html)。

---

## 1. 目標環境（本手冊預設）

| 項目 | 規格 |
|------|------|
| Hypervisor | VMware ESXi **8.0 Update 3 P06 或更新的 8.0 update**（單機）<br>⚠️ NVIDIA vGPU 20.0 / 20.1 的 [VMware vSphere Release Notes](https://docs.nvidia.com/vgpu/20.0/grid-vgpu-release-notes-vmware-vsphere/index.html) 要求 ESXi **8.0u3 P06 and later updates**；早期的 8.0U3 build 不在支援範圍 |
| 管理平台 | VMware vCenter Server **8.0 Update 3**（VCSA），對應 ESXi 8.0U3 P06+ build |
| VMware 授權 edition | **vSphere Foundation** 或 **vSphere Enterprise Plus**（vGPU on ESXi 需要） |
| 實體 GPU | **NVIDIA H200 NVL × 2**（PCIe 141 GB form factor） |
| NVAIE Infra 版本 | **8.1**（R595 Driver / vGPU 20.1）|
| GPU 模式 | **vGPU**（非 Passthrough，需 VIB + DLS）|
| Guest OS | **Ubuntu Server 24.04 LTS**（amd64） |
| 測試 VM 數 | **2 台**，名稱 `vgpu-test-01` / `vgpu-test-02` |
| 授權方式 | **DLS**（Delegated License Service，地端 License Server）|

> 若使用 NVAIE 7.5 LTSB：Driver 為 R580、vGPU Software 為 19.5，其餘流程相同。版本選擇說明見 §3。

---

## 2. 整體建置流程（高階）

```
[階段 0] 先決條件確認與軟體取得
                ↓
[階段 1] ESXi 8.0U3 + vCenter 8.0U3 主機環境就緒
                ↓
[階段 2] 在 ESXi Host 安裝 NVAIE Host Software（VIB）並設為 Shared Direct
                ↓
[階段 3] 部署 DLS License Server VM，產生 Client Configuration Token
                ↓
[階段 4] 建立 2 個 Ubuntu 24.04 測試 VM，指派 vGPU Profile
                ↓
[階段 5] 安裝 Ubuntu 24.04 Guest OS + vGPU Guest Driver（-grid）
                ↓
[階段 6] 在 Guest 套用 Client Configuration Token，取得授權
                ↓
[階段 7] 雙 VM + ESXi Host 端 vGPU 狀態驗證
```

---

## 3. 文件清單

| 文件 | 涵蓋內容 |
|------|---------|
| [00-README.md](00-README.md) | 本文件 — 索引、目標、流程地圖、版本決策 |
| [01-prerequisites.md](01-prerequisites.md) | 階段 0／1：硬體/BIOS、軟體下載、版本相容性、網路規劃 |
| [02-esxi-host-vib.md](02-esxi-host-vib.md) | 階段 2：VIB 上傳、安裝、Graphics Type 設定、Host 端 `nvidia-smi` 驗證 |
| [03-dls-license-server.md](03-dls-license-server.md) | 階段 3：DLS VM 部署、Service Instance 建立、Client Token 產生 |
| [04-vm-creation-and-vgpu-profile.md](04-vm-creation-and-vgpu-profile.md) | 階段 4：建立 2 個 Ubuntu 24.04 VM、指派 vGPU Profile、MMIO/EFI 設定 |
| [05-guest-os-and-driver.md](05-guest-os-and-driver.md) | 階段 5／6：Ubuntu 安裝、Grid Driver 安裝、Token 套用 |
| [06-validation.md](06-validation.md) | 階段 7：雙 VM Guest 端 + ESXi Host 端完整驗證指令與檢查清單 |
| [07-troubleshooting.md](07-troubleshooting.md) | 常見錯誤排查（VIB 安裝、`insufficient graphics resources`、授權未發放、Driver 編譯失敗等） |

---

## 4. 關鍵決策：NVAIE Infra 版本選擇

| NVAIE 版本 | Data Center Driver | vGPU Software | 支援到 | vSphere 8.0 and later | 適用情境 |
|-----------|-------------------|---------------|--------|----------------------|---------|
| **Infra 8.1**（Feature） | 595.71.05 | 20.1 | 2027/04 | ✅ | **本手冊主例**；想用最新功能與最新 driver |
| Infra 7.5 LTSB | 580.159.03 | 19.5 | 2028/07 | ✅ | 偏好長期穩定（3 年支援），可作替代 |
| Infra 7.2 | — | — | Archived | — | ❌ 不要用於新建置 |
| Infra 6.x | — | — | EOL（2026/03） | — | ❌ 已 EOL |

> ESXi 8.0U3 屬於官方 Support Matrix 的「`8.0 and later`」範圍，**Infra 8.1 與 7.5 LTSB 都支援**。詳細依各版本當下 Support Matrix 原文為準。
>
> ⚠️ **仍須滿足 P06+ caveat：** NVAIE Support Matrix 的「8.0 and later」是泛指版號，**並未豁免** NVIDIA vGPU 20.0/20.1 VMware vSphere Release Notes 對 ESXi build 的要求（**8.0u3 P06 and later updates**）。本手冊目標環境（§1）與先決條件（[01-prerequisites.md](01-prerequisites.md) §3）皆已要求 P06+，此處表格僅是版號相容性歸納，實際 build 仍需符合 P06+。
>
> 來源：[Support Matrix 8.1](https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html)、[Support Matrix 7.5 LTSB](https://docs.nvidia.com/ai-enterprise/release-7/7.5/support/support-matrix.html)、[NVIDIA Virtual GPU Software for VMware vSphere Release Notes](https://docs.nvidia.com/vgpu/20.0/grid-vgpu-release-notes-vmware-vsphere/index.html)

---

## 5. 本手冊範例 vGPU Profile 分派（H200 NVL）

H200 NVL（PCIe 141 GB）官方 C-series time-sliced profile 共 9 個（見 `04` 文件 §3.1 完整表）：

```
H200-141C / H200-70C / H200-35C / H200-28C / H200-17C / H200-14C / H200-8C / H200-7C / H200-4C
```

本手冊預設範例：**兩個 VM 各取一張完整 GPU 的一半**

| 測試 VM | 指派 vGPU Profile | 框緩衝記憶體（官方 profile 容量） | 來源實體 GPU | 占用比例 |
|---------|-------------------|--------------------------------|-------------|---------|
| `vgpu-test-01` | `nvidia_h200-70c` | **70 GB** | H200 NVL #0 | 1/2 |
| `vgpu-test-02` | `nvidia_h200-70c` | **70 GB** | H200 NVL #1 | 1/2 |

> Profile 容量以官方 [Hopper Architecture vGPU Types](https://docs.nvidia.com/ai-enterprise/release-8/latest/infra-software/vgpu/reference/hopper.html) 為準（H200-70C = **70 GB**、H200-141C = **141 GB**）。Guest 端 `nvidia-smi` 顯示的 MiB 數（如 H200-70C ≈ 72704 MiB）為實際呈現給 OS 的 Frame Buffer，與官方 profile GB 數略有換算差異，屬正常。

> **為何選 `H200-70C`：** 既能展示 vGPU 切分（1 張 H200 NVL = 2 個 vGPU），又能讓兩個 VM 分別落在不同實體 GPU 上各自佔半張，留下另一半作日後擴充或單實體 GPU 雙 vGPU 共享測試。
>
> **替代方案：**
> - 想驗證「2 個 VM 共享同一張 H200 NVL」 → 兩個 VM 都用 `H200-70C`，並透過 `pciPassthru0.cfg.gpu-pci-id` 強制落在 GPU #0。
> - 想驗證「整張 GPU 給單一 VM」 → 兩個 VM 各用 `H200-141C`，分別落在 #0 / #1。
> - 想驗證更細的切分（MIG-backed） → 使用 `H200-1-18C`（最多 7 vGPU/GPU）等 MIG profile，需先在 Host 端啟用 MIG 模式。
>
> **vGPU 同 GPU 同 profile 規則：** 預設模式（homogeneous）下，**同一張實體 GPU 上的所有 time-sliced C-series vGPU 必須使用相同的 profile**。NVAIE 8.1（vGPU 20.x）支援 heterogeneous 模式，但需額外啟用，本手冊不採用。

---

## 6. 預期完成里程碑

完成本手冊後，應可達到以下狀態：

- ESXi Host：`nvidia-smi` 顯示兩張 H200 NVL，`nvidia-smi vgpu` 列出兩個 vGPU 實例。
- ESXi Host：Graphics Type 顯示為 `Shared Direct`。
- DLS：管理介面顯示兩筆 vGPU client lease（Token 套用後）。
- VM #1（`vgpu-test-01`）：`nvidia-smi` 顯示一張 `NVIDIA H200-70C`，官方 profile 容量 **70 GB**（`nvidia-smi` 可能顯示約 72704 MiB）；`nvidia-smi -q | grep "License"` 顯示 `Licensed`。
- VM #2（`vgpu-test-02`）：同上。
- 兩個 VM 都能執行 `nvidia-smi -q -x` 不報錯，可進行下游 CUDA / Docker GPU 工作負載。

詳細驗證指令與檢查清單見 [06-validation.md](06-validation.md)。

---

## 7. 主要參考連結

| 主題 | URL |
|------|-----|
| NVAIE VMware 部署主頁 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/index.html |
| 先決條件 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/prereqs.html |
| Host Software（VIB） | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/vgpu.html |
| 建立第一台 VM | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/first-vm.html |
| NVIDIA License System | https://docs.nvidia.com/license-system/latest/ |
| H200 NVL vGPU Types Reference（8.1） | https://docs.nvidia.com/ai-enterprise/release-8/latest/infra-software/vgpu/reference/hopper.html |
| Support Matrix 8.1 | https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html |
| NGC（軟體下載） | https://ngc.nvidia.com |
| NVIDIA Licensing Portal | https://licensing.nvidia.com |
