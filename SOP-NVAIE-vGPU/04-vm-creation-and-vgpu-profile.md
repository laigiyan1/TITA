# 04 — 建立 2 個 Ubuntu 24.04 VM 並指派 vGPU Profile

> 對應流程：階段 4
> 完成本章節後，vCenter 上會有 `vgpu-test-01` 與 `vgpu-test-02` 兩個 VM，皆使用 EFI Firmware、設定好 64-bit MMIO、分別綁定一個 `nvidia_h200-70c` PCI Device，並掛載 Ubuntu 24.04 ISO 準備開機安裝。
> Guest OS 安裝與 Grid Driver 安裝在 [05-guest-os-and-driver.md](05-guest-os-and-driver.md) 處理。

---

## 1. 章節前置條件

- [02-esxi-host-vib.md](02-esxi-host-vib.md) 全部 ✅（VIB 已裝、Shared Direct 已切換、兩張 H200 NVL 在 Host 端 `nvidia-smi` 看得到）。
- [03-dls-license-server.md](03-dls-license-server.md) 全部 ✅（DLS 已上線，Token 已下載）。
- Ubuntu 24.04 ISO 已上傳到 ESXi Datastore 或可掛載至 VM。

---

## 2. H200 NVL vGPU Profile 速查

> 來源：[H200 vGPU Types Reference (NVAIE 8.1)](https://docs.nvidia.com/ai-enterprise/release-8/latest/infra-software/vgpu/reference/hopper.html) — 對應 NVIDIA H200 PCIe 141GB（H200 NVL）。
> 在 vCenter Web Client 的 NVIDIA GRID vGPU 下拉選單中，profile 通常顯示為 `nvidia_h200-141c` / `nvidia_h200-70c` 等小寫格式；NVIDIA 文件採大寫 `H200-141C` 表記法，**兩者等價**。

### 2.1 Time-Sliced（C-series，預設）

| Profile（vCenter 顯示） | 文件名稱 | 框緩衝（官方 profile 容量） | 每張 GPU 最大 vGPU 數 | 典型用途 |
|------------------------|---------|---------------------------|---------------------|---------|
| `nvidia_h200-141c` | H200-141C | **141 GB** | 1 | 整張 GPU 給單一 VM |
| `nvidia_h200-70c` | H200-70C | **70 GB** | 2 | **本手冊預設**：兩 VM 各佔 1/2 GPU |
| `nvidia_h200-35c` | H200-35C | **35 GB** | 4 | 中型推論 |
| `nvidia_h200-28c` | H200-28C | **28 GB** | 5 | |
| `nvidia_h200-17c` | H200-17C | **17 GB** | 8 | |
| `nvidia_h200-14c` | H200-14C | **14 GB** | 10 | |
| `nvidia_h200-8c` | H200-8C | **8 GB** | 16 | |
| `nvidia_h200-7c` | H200-7C | **7 GB** | 20 | |
| `nvidia_h200-4c` | H200-4C | **4 GB** | 32 | 最小切分 |

> 上表 GB 數為 [Hopper Architecture vGPU Types](https://docs.nvidia.com/ai-enterprise/release-8/latest/infra-software/vgpu/reference/hopper.html) 官方 profile 規格。Guest 端 `nvidia-smi` 顯示的 MiB 數（如 H200-70C ≈ 72704 MiB）為實際暴露給 OS 的 Frame Buffer，與官方 profile GB 數略有換算差異屬正常。

### 2.2 MIG-Backed（需先在 Host 啟用 MIG，本手冊不採用）

| Profile | 框緩衝 | 每張 GPU 最大 vGPU 數 |
|---------|--------|---------------------|
| `nvidia_h200-7-141c` | 141 GB | 1 |
| `nvidia_h200-4-71c` | 71 GB | 1 |
| `nvidia_h200-3-71c` | 71 GB | 2 |
| `nvidia_h200-2-35c` | 35 GB | 3 |
| `nvidia_h200-1-35c` | 35 GB | 4 |
| `nvidia_h200-1-18c` / `nvidia_h200-1-18cme` | 18 GB | 7 / 1 |

> **本手冊預設不啟用 MIG**：兩個 VM 都使用 time-sliced `nvidia_h200-70c`，分別落在不同實體 GPU 上。如需 MIG，請先在 ESXi Host 上 `nvidia-smi -i <gpu> -mig 1`，再選 MIG-backed profile（詳見 `../doc-NVAIE/06-NVAIE-Advanced-GPU-Config.md` §5）。

---

## 3. VM 規格設計

本手冊兩個測試 VM 採相同規格：

| 資源 | 設定值 | 說明 |
|------|--------|------|
| Guest OS | Ubuntu Linux 64-bit | 對應 Ubuntu 24.04 LTS |
| Boot Firmware | **EFI** | C-series vGPU **必須** |
| Secure Boot | 停用 | 簡化 Grid Driver `.run` 安裝（章節 05） |
| vCPU | 16（1 socket × 16 cores） | |
| RAM | 64 GB | 預留 Reservation = 64 GB（vGPU **強制** 全部記憶體預留）|
| Disk | 150 GB（Thin Provisioned） | 系統 + Docker workload 預留 |
| 網卡 | VMXNET3 × 1 | 連到 DLS 同一網段 |
| vGPU Profile | `nvidia_h200-70c` | 官方 profile 容量 **70 GB** 框緩衝（`nvidia-smi` 可能顯示約 72704 MiB） |
| 進階參數 | `pciPassthru.use64bitMMIO=TRUE`, `pciPassthru.64bitMMIOSizeGB=128` | H200 NVL 單卡建議 128 GB MMIO |

> **記憶體預留：** vGPU C-series profile 規範 VM 開機時必須有等於 RAM 大小的記憶體預留，否則 VM 無法啟動（`Reservation: Reserve all guest memory`）。本手冊規格 RAM = 64 GB 表示 Reservation 也 = 64 GB。

> **MMIO 值（H200 NVL）：** NVIDIA 官方 `first-vm.html` 的 MMIO 對照表目前**未列出 H200**，業界部署慣例 H100/H200 單卡 ≥ 128 GB。若 VM 開機出現 `Module DevicePowerOn failed`，請將值向上取 2 的次方（256、512、1024 GB）。
> 來源：[en.vmik.net — Adjusting MMIO values in ESXi 8U3](https://en.vmik.net/2025/10/vmw-esxi-adjusting-mmio/)、[NVIDIA AIE first-vm 文件](https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/first-vm.html)。

---

## 4. 建立 VM #1：`vgpu-test-01`

以下為 vCenter Web Client 操作步驟。

### 4.1 New Virtual Machine 精靈

1. 右鍵你的 Cluster / Host → **New Virtual Machine**。
2. **Select a creation type**：`Create a new virtual machine`。
3. **Select a name and folder**：
   - VM name：`vgpu-test-01`
   - 目標 Datacenter / Folder
4. **Select a compute resource**：你的 ESXi 8.0U3 Host（內含 H200 NVL）。
5. **Select storage**：選 Datastore（容量 ≥ 200 GB），Disk Format 選 **Thin Provision**。
6. **Select compatibility**：`ESXi 8.0 U2 and later`（VM Hardware version 21 或更新；可選最新相容版本）。
7. **Select a guest OS**：
   - Guest OS Family：`Linux`
   - Guest OS Version：`Ubuntu Linux (64-bit)`
8. **Customize hardware**（保留在此頁先做完進階設定，再下一步）：

### 4.2 Customize Hardware — Virtual Hardware 標籤

| 項目 | 設定值 |
|------|--------|
| CPU | 16 |
| Cores per Socket | 16（1 socket） |
| CPU Reservation | （可留 0） |
| Memory | 65536 MB（64 GB） |
| Memory **Reservation** | **65536 MB**（勾選 `Reserve all guest memory (All locked)`） |
| New Hard disk | 150 GB（Thin） |
| New Network → Adapter Type | VMXNET3，連到目標 Port Group |
| New CD/DVD Drive | **Datastore ISO File** → 選 Ubuntu 24.04 ISO；勾選 **Connect at power on** |
| **移除** Floppy / USB controller（選用，清理裝置） | |

### 4.3 Customize Hardware — VM Options 標籤

1. **Boot Options → Firmware**：選 **EFI**。
2. **Boot Options → Secure Boot**：**取消勾選**（本手冊不啟用 Secure Boot）。
3. **Advanced → Edit Configuration**：依下表新增兩筆參數（之後 §4.5 也會用到）：

| Key | Value |
|-----|-------|
| `pciPassthru.use64bitMMIO` | `TRUE` |
| `pciPassthru.64bitMMIOSizeGB` | `128` |

4. 完成 → **Next** → **Finish**。

> 此時 VM 已建立但**還沒有 PCI Device（vGPU）**，下一步加入。

### 4.4 加入 vGPU（New PCI Device）

1. 在 VM 上 → 右鍵 **Edit Settings**。
2. **ADD NEW DEVICE → PCI Device**（或 `Shared PCI Device`，依 vCenter 版本字樣）。
3. 出現 New PCI device 列：
   - **GPU Profile**：選 `nvidia_h200-70c`（如選單未列出 H200 → 表示 VIB 未生效或 Graphics Type 未切換為 Shared Direct，回 02 §6/§7）
   - **Reserve all memory** 應自動勾選且鎖住（不可改）
4. 點 **OK** 儲存。

### 4.5 確認進階參數（再次檢查）

回到 VM → **Edit Settings → VM Options → Advanced → Edit Configuration**，確認以下三筆都存在：

| Key | Value | 來源 |
|-----|-------|------|
| `pciPassthru.use64bitMMIO` | `TRUE` | 手動加入（§4.3） |
| `pciPassthru.64bitMMIOSizeGB` | `128` | 手動加入（§4.3） |
| `pciPassthru0.cfg.gpu-pci-id` | （自動產生，留空或 vCenter 自動填入） | 加入 PCI Device 後自動處理 |

> **強制指定 GPU 位置**（選用）：若想固定本 VM 必須使用 H200 NVL #0（PCI BDF 例：`0000:31:00.0`），加入：
>
> ```
> pciPassthru0.cfg.gpu-pci-id = "0000:31:00.0"
> ```
>
> BDF 從 ESXi `lspci | grep -i nvidia` 取得。預設不強制指定時，vCenter 會依 *Shared passthrough GPU assignment policy*（章節 02 §7.1）自動分配。

---

## 5. 建立 VM #2：`vgpu-test-02`

完全比照 §4，只變更：

| 欄位 | 值 |
|------|---|
| VM name | `vgpu-test-02` |
| 其餘規格與進階參數 | 與 `vgpu-test-01` **完全相同**（同樣 16 vCPU / 64 GB RAM / 150 GB / VMXNET3 / EFI / 128 GB MMIO / `nvidia_h200-70c`） |

**選用：強制落在 H200 NVL #1**（讓兩 VM 各佔一張 GPU，章節 00 §5 預設行為）

```
pciPassthru0.cfg.gpu-pci-id = "0000:CA:00.0"
```

> 不強制指定時，因 *Shared passthrough GPU assignment policy = Spread VMs across GPUs*，且兩個 VM 都使用 `nvidia_h200-70c`（每張 GPU 最多 2 個此 profile），通常會自動把第二個 VM 排到另一張 GPU。

---

## 6. 確認兩個 VM 已就緒（尚未開機）

在 vCenter Inventory 應該看到：

```
DC-LAB
└── CL-NVAIE
    └── <esxi-host>
        ├── dls-licsrv-01   ✅ Powered On
        ├── vgpu-test-01    ⛔ Powered Off
        │   ├── 16 vCPU / 64 GB RAM (reserved) / 150 GB HDD
        │   ├── PCI Device: NVIDIA GRID vGPU nvidia_h200-70c
        │   └── Boot Firmware: EFI
        └── vgpu-test-02    ⛔ Powered Off
            ├── 16 vCPU / 64 GB RAM (reserved) / 150 GB HDD
            ├── PCI Device: NVIDIA GRID vGPU nvidia_h200-70c
            └── Boot Firmware: EFI
```

---

## 7. 開機並確認 VM 能正常啟動（不需安裝 OS）

> 這一步是**驗證 vGPU 指派沒有設定錯誤**，避免在裝完 OS 才發現 MMIO 不足或 Profile 衝突。

1. 對 `vgpu-test-01` 按 **Power On**。
2. 觀察 vCenter Recent Tasks，應顯示成功 `Power On virtual machine`，無錯誤訊息。
3. 等 30 秒 → 開啟 **Launch Web Console**，應看到 Ubuntu installer 的開機選單（按 Enter 進入 GRUB）。
4. 暫時不安裝 OS，先 **Power Off** VM。
5. 對 `vgpu-test-02` 重複步驟 1–4。

### 7.1 若開機失敗常見錯誤

| 錯誤 | 原因 | 處理 |
|------|------|------|
| `Module DevicePowerOn failed` | MMIO 太小 | 將 `pciPassthru.64bitMMIOSizeGB` 提升到 256 → 512 → 1024 |
| `Insufficient graphics resources` | Graphics Type 還是 Shared | 回 02 §7 重新切換並重啟服務 |
| `Unable to retrieve the host GPU information` | VIB 未生效 / Host 未重開機 | 回 02 §4–§6 |
| `The specified vGPU type ... is not supported` | Profile 與其他 VM 衝突（同 GPU 上有不同 profile） | 改成同 profile 或落到另一張 GPU |

---

## 8. 完成檢查清單

- [ ] `vgpu-test-01` 已建立，Firmware = EFI，MMIO `use64bitMMIO=TRUE` 與 `64bitMMIOSizeGB=128` 已設定
- [ ] `vgpu-test-01` 已掛載 `nvidia_h200-70c` PCI Device
- [ ] `vgpu-test-01` 記憶體預留 = 64 GB（lock all guest memory）
- [ ] `vgpu-test-02` 同上規格、同上設定
- [ ] 兩個 VM 都能 Power On 不報錯（停在 Ubuntu installer 啟動畫面即可）
- [ ] 已下載並準備好兩個 VM 用的 Grid Driver `.run` 與 DLS Token `.tok`

→ 全部 ✅ 後，前往 [05-guest-os-and-driver.md](05-guest-os-and-driver.md)。
