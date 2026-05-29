# NVAIE 進階 GPU 設定

> 來源：https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/advance-gpu.html  
> 整理日期：2026-05-20

---

## 1. GPU 分割方式

### 1.1 時間分割（Temporal Partitioning / Time-Slicing）
- **軟體層分割**，GPU 資源輪流分配
- 每張 GPU 可同時承載的 vGPU 數量依 profile 大小而定：H100 PCIe 使用最小 profile（`H100-4C`）最多可達 **20 個 VM**（來源：[Hopper vGPU Types Reference](https://docs.nvidia.com/ai-enterprise/release-7/7.5/infra-software/vgpu/reference/hopper.html)）
- 平衡分配圖形資源
- 適合非即時工作負載

### 1.2 空間分割（MIG - Multi-Instance GPU）
- **硬體層分割**，真正隔離的 GPU 實例
- 每個 MIG 實例有獨立的：Streaming Multiprocessors、Memory Controller、Cache
- **A100 / H100 支援 MIG**
- 適合需要隔離保證的多租戶環境

---

## 2. vGPU 排程策略

透過 ESXi 指令設定，影響多 VM 共享 GPU 時的資源分配：

### 2.1 設定所有 GPU 的排程策略

```bash
# 登入 ESXi Host，執行：
esxcli system module parameters set -m nvidia -p \
  "NVreg_RegistryDwords=RmPVMRL=<value>"
```

### 2.2 排程策略數值

| Registry 值 | 策略 | 說明 |
|-------------|------|------|
| `0x00` | Best Effort（預設） | Round-robin，最大密度，適合不同負載 |
| `0x01` | Equal Share（預設 timeslice） | 依活躍 VM 數量動態分配 |
| `0x11` | Fixed Share（預設 timeslice） | 每個 vGPU 保證固定 QoS |
| `0x00TT0001` | Equal Share（自訂 timeslice） | TT 為 hex 值（01-1E ms） |

### 2.3 設定特定 GPU 的排程策略

```bash
# 先查詢 GPU 的 PCI 位址
lspci | grep NVIDIA

# 針對特定 GPU 設定
esxcli system module parameters set -m nvidia -p \
  "NVreg_RegistryDwordsPerDevice=pci=<domain:bdf>;RmPVMRL=<value>"
```

---

## 3. 多 GPU VM 設定（GPU 聚合）

### 3.1 HGX 多 GPU VM 部署模式

**Full PCIe Passthrough（整板透傳）**  
將整張 HGX baseboard 的多顆 GPU 以 Passthrough 方式指派給單一 VM。適合需要整板效能、高效能訓練，或不使用 vGPU 授權的場景。多 GPU Passthrough 必須手動指定在推薦的 NUMA 節點上。

**vGPU（VMware vSphere 上）**  
依據 NVIDIA Support Matrix，在 VMware vSphere 上，HGX 平台的 NVIDIA vGPU for Compute 可支援 **1/2/4/8 GPU per VM** 的配置。實際可用組合須依以下條件確認：
- 部署的 NVAIE Infra 版本（7.5 LTSB / 8.1）
- VMware vSphere 版本
- GPU 平台（H100 SXM5 / H200 SXM5）
- NVIDIA vGPU software 版本

> **其他 Hypervisor（KVM 等）：** 不同 Hypervisor 的 HGX 多 GPU 支援條件各異，不可套用 VMware vSphere 的支援情境。請查閱對應 Hypervisor 的 NVIDIA/廠商支援文件。

### 3.2 查詢 GPU 拓撲
```bash
# 在 ESXi Host 上執行
nvidia-smi topo -m
```
此指令顯示哪些 GPU 透過 NVLink 互連，用於確認可以做 GPU-to-GPU 直接資料傳輸的組合。

### 3.3 多 GPU VM 進階設定參數（VM .vmx 檔）

```
pciPassthru0.cfg.gpu-pci-id = "ssss:bb:dd.f"
pciPassthru1.cfg.gpu-pci-id = "ssss:bb:dd.f"
```

其中 `ssss:bb:dd.f` 為 GPU 的 PCI BDF 位址（透過 `lspci | grep NVIDIA` 取得）。

### 3.4 NUMA 考量
- 對於 4 個或以上 GPU 的配置，NUMA 節點設定對性能至關重要
- 建議將 VM 的 vCPU 和記憶體固定到與 GPU 相同的 NUMA 節點

---

## 4. vSphere 8 Device Groups（設備群組）

vSphere 8 新功能，簡化多 GPU 指派：

- 可將共享 PCIe Switch 或透過 NVLink 互連的多張 GPU 組成一個 Device Group
- 透過 vSphere 標準 PCI 設備工作流程一次指派整組 GPU 給 VM
- 自動支援 DRS 和 HA 感知

---

## 5. MIG（Multi-Instance GPU）設定

### 5.1 H100 MIG 支援的分割

H100 80GB 可分割為：

| MIG Profile | GPU 實例數 | 記憶體 | 適用場景 |
|------------|-----------|--------|----------|
| 1g.10gb | 7個 | 10 GB | 小型推論 |
| 2g.20gb | 3個 | 20 GB | 中型推論 |
| 3g.40gb | 2個 | 40 GB | 大型推論 |
| 4g.40gb | 1個 | 40 GB | 訓練 |
| 7g.80gb | 1個 | 80 GB | 整張 GPU |

### 5.2 啟用 MIG 模式（在 ESXi Host 上）

```bash
# 啟用 MIG（-i 指定 GPU ID）
nvidia-smi -i 0 -mig 1

# GPU Reset（啟用 MIG 後需要）
nvidia-smi -i 0 --gpu-reset

# 確認 MIG 已啟用
nvidia-smi -i 0 --query-gpu=pci.bus_id,mig.mode.current --format=csv
```

### 5.3 建立 MIG GPU 實例

```bash
# 列出可用的 MIG GPU Instance Profiles
nvidia-smi mig -lgip

# 建立 GPU 實例（-gi 指定 profile ID）
nvidia-smi mig -cgi <profile-id>

# 列出已建立的 GPU 實例
nvidia-smi mig -lgi
```

### 5.4 建立 Compute Instance

```bash
# 列出 Compute Instance Profiles
nvidia-smi mig -lcip

# 在 GPU 實例上建立 Compute Instance
nvidia-smi mig -cci -gi <gpu-instance-id>
```

### 5.5 在 vSphere 中指派 MIG vGPU

1. 編輯 VM 設定 → Add New Device → PCI Device
2. 選擇 MIG-backed vGPU Profile（命名格式通常為 `<GPU>-<GI slices>-<memory>C`；部分 media-extension profile 會以 `CME` 結尾，例如 `H200-1-18CME`、`H100XM-1-10CME`）

MIG-backed vGPU Profile 正式命名範例：

| GPU | MIG-backed Profile 範例 | 說明 |
|-----|------------------------|------|
| H100 PCIe | `H100-7-80C`、`H100-4-40C`、`H100-1-10C` | 數字代表 MIG GI 大小 |
| H100 SXM5 | `H100XM-7-80C`、`H100XM-2-20C`、`H100XM-1-10C` | SXM5 用 XM 後綴 |
| H200 PCIe (NVL) | `H200-7-141C`、`H200-4-71C`、`H200-1-18C`、`H200-1-18CME` | PCIe/NVL 用 H200-；`H200-1-18C`（7 vGPU/GPU）與 `H200-1-18CME`（media-extension）均存在 |
| H200 SXM5 | `H200X-7-141C`、`H200X-4-71C`、`H200X-1-18CME` | SXM5 用 H200X- |

> 請勿使用 `grid_h100-*` 這類舊式小寫格式，NVAIE 7.x/8.x vGPU Types Reference 已改用上述格式。

---

## 6. ECC 記憶體管理

```bash
# 停用 ECC（所有 GPU）
nvidia-smi -e 0

# 停用 ECC（指定 GPU）
nvidia-smi -i 0000:02:00.0 -e 0

# 啟用 ECC（所有 GPU）
nvidia-smi -e 1

# 啟用 ECC（指定 GPU）
nvidia-smi -i 0000:02:00.0 -e 1
```

> **注意**：ECC 設定更改後需要重開機（Host 或 VM）才能生效。

---

## 7. H100 HGX 多 GPU 特殊注意事項

### 7.1 NVLink 拓撲
- H100 SXM5 透過 NVLink 4.0 互連
- HGX H100 8-GPU：所有 8 GPU 在同一 NVSwitch 網格內
- `nvidia-smi topo -m` 可查看實際連接關係

### 7.2 多 GPU Passthrough VM 建議配置
- GPU per VM 數量依 NUMA 拓撲與 PCIe 拓撲規劃；超過 4 GPU per VM 需特別注意 NUMA 對齊
- 確保 VM vCPU 和記憶體固定在與 GPU 相同的 NUMA 節點（來源：[NVIDIA Advance GPU Configuration](https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/advance-gpu.html)）

### 7.3 HGX vGPU 部署說明
- **VMware vSphere 上**：HGX 平台的 vGPU for Compute 可支援 1/2/4/8 GPU per VM；Full PCIe Passthrough 亦是整板透傳的重要選項（適合高效能訓練或需要整板獨佔）。
- **其他 Hypervisor**：HGX 平台支援情況不同，需查對應廠商文件，不可套用 VMware vSphere 的支援條件。
- 單一 H100 SXM5 可用 vGPU time-slicing（最多 20 個 vGPU 實例，依 profile 大小而定，例如 `H100XM-4C` 最多 20 個）。

---

## 8. 參考連結

| 文件 | URL |
|------|-----|
| 進階 GPU 設定 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/advance-gpu.html |
| MIG User Guide | https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html |
| vGPU 排程說明 | https://docs.nvidia.com/grid/latest/grid-vgpu-user-guide/index.html |
| H100/H200 vGPU Types Reference (7.5) | https://docs.nvidia.com/ai-enterprise/release-7/7.5/infra-software/vgpu/reference/hopper.html |
| Support Matrix 8.1 | https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html |
| Support Matrix 7.5 | https://docs.nvidia.com/ai-enterprise/release-7/7.5/support/support-matrix.html |
