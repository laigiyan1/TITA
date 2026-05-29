# 02 — ESXi Host：安裝 NVAIE Host Software（VIB）並切換 Shared Direct

> 對應流程：階段 2
> 完成本章節後，ESXi 8.0U3 主機已載入 NVIDIA 核心模組、`nvidia-smi` 在 Host 端可顯示兩張 H200 NVL，Graphics Type 已切換為 `Shared Direct`，準備好接受 vGPU VM。

---

## 1. 章節前置條件

- ESXi 8.0U3 已可登入；vCenter 8.0U3 已將該 Host 納管。
- ESXi 上 **SSH** 與 **ESXi Shell** 服務已啟用。
- 已下載 NVAIE Infra 8.1 對應的 `NVIDIA-AIE_ESXi_*.vib`（章節 01 §5）。
- Host 上**所有 VM 已關機**（VIB 安裝必須進入維護模式，無 VM 可存活）。

---

## 2. 上傳 VIB 至 ESXi Datastore

> 兩種方式擇一。建議方式 A（vCenter Web Client），讀者較熟悉。

### 方式 A：vCenter Web Client

1. 登入 vCenter（`https://<vcenter>/ui`）。
2. 左側 **Inventory** → 找到你的 ESXi Host → 切到 **Datastores** 標籤。
3. 點選一個有足夠空間（≥ 500 MB）的本地 Datastore（例：`datastore1`） → 右鍵 → **Browse Files**。
4. 點 **New Folder**，命名為 `VIB`。
5. 進入 `VIB` 資料夾 → **Upload Files** → 選擇 `NVIDIA-AIE_ESXi_*.vib` 上傳。

### 方式 B：SCP

```powershell
# 在管理機（Windows PowerShell）以 OpenSSH client 上傳
scp .\NVIDIA-AIE_ESXi_<version>.vib root@<esxi-ip>:/vmfs/volumes/datastore1/VIB/
```

> 上傳完成後 SSH 進 ESXi，確認檔案存在：
> ```bash
> ls -lh /vmfs/volumes/datastore1/VIB/
> ```

---

## 3. 將 ESXi Host 進入 Maintenance Mode

在 vCenter：右鍵 Host → **Maintenance Mode → Enter Maintenance Mode**；或 SSH 執行：

```bash
esxcli system maintenanceMode set --enable=true
```

> 確認執行前，Host 上的 VM **全部** 已關機或已遷出（單機環境無 vMotion 目的地，請直接關機）。

---

## 4. 安裝 VIB

SSH 登入 ESXi 主機（`ssh root@<esxi-ip>`），執行：

```bash
# 一定要用 /vmfs/volumes/ 絕對路徑，不要用 ds:/// 格式
esxcli software vib install -v /vmfs/volumes/datastore1/VIB/NVIDIA-AIE_ESXi_<version>.vib
```

### 4.1 預期輸出（成功）

```
Installation Result
   Message: Operation finished successfully.
   Reboot Required: false
   VIBs Installed: NVIDIA_bootbank_NVIDIA-AIE_ESXi_8.0.0_Driver_<version>
   VIBs Removed:
   VIBs Skipped:
```

### 4.2 重要：**即使顯示 `Reboot Required: false`，仍必須重開機**

NVIDIA 文件明確要求重開機以載入核心模組與啟動 `nv-hostengine` / Xorg：

```bash
reboot
```

> 預估重開機 ≈ 5 分鐘（取決於伺服器 BIOS POST 與 H200 NVL 初始化時間）。

---

## 5. 離開 Maintenance Mode

重開機完成後 SSH 進 ESXi：

```bash
esxcli system maintenanceMode set --enable=false
```

或在 vCenter 右鍵 Host → **Exit Maintenance Mode**。

---

## 6. 驗證 VIB 安裝與 GPU 可見性

SSH 進 ESXi 後執行以下三個指令，**三者都通過**才算成功：

### 6.1 確認 VIB 已掛載

```bash
esxcli software vib list | grep -i nvidia
```

預期：列出 `NVIDIA-AIE_ESXi_8.0.0_Driver_<version>`，狀態欄為 `VMwareAccepted` 或 `VMwareCertified`。

### 6.2 確認核心模組已載入

```bash
vmkload_mod -l | grep nvidia
```

預期：列出 `nvidia` 模組，Loaded 欄為 `Yes`。

### 6.3 確認兩張 H200 NVL 都可見

```bash
nvidia-smi
```

預期輸出（樣式示意，版本號以實際 driver 顯示為準；R595 / 595.71.05 預期會顯示 `CUDA Version: 13.x`）：

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.71.05            Driver Version: 595.71.05    CUDA Version: 13.x         |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                                | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap  |           Memory-Usage | GPU-Util  Compute M. |
|                                          |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H200 NVL                Off  |   00000000:31:00.0 Off |                    0 |
| N/A   34C    P0             71W / 600W   |       0MiB / 144384MiB |      0%      Default |
|                                          |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA H200 NVL                Off  |   00000000:CA:00.0 Off |                    0 |
| N/A   33C    P0             70W / 600W   |       0MiB / 144384MiB |      0%      Default |
|                                          |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
```

> 兩張卡都應該顯示 `NVIDIA H200 NVL`，官方 HBM3e 容量 **141 GB**（`nvidia-smi` 會顯示 `144384 MiB`，屬 GiB↔GB 換算差異）。若只看到一張或型號為 `Unknown` → 直接前往 [07-troubleshooting.md](07-troubleshooting.md) §1。

### 6.4 確認 GPU 拓撲（選用）

```bash
nvidia-smi topo -m
```

預期：兩張 GPU 之間若有 NVLink 橋接器，會顯示 `NV*` 字樣；無 NVLink 則顯示 `SYS` / `PHB`。H200 NVL 多卡通常以 PCIe / NVLink Bridge 連接，視伺服器拓撲。

---

## 7. 切換 Graphics Type 為 `Shared Direct`（vGPU 必要步驟）

**ESXi 預設 Graphics Type 為 `Shared`（vSGA 模式，不支援 vGPU）**。必須改為 `Shared Direct`，否則 vGPU VM 開機時會出現：

```
Insufficient graphics resources available on host
```

### 7.1 透過 vCenter Web Client 變更

1. 登入 vCenter → 選 ESXi Host → **Configure** 標籤。
2. 左側選單 → **Hardware → Graphics**。
3. 切到 **Host Graphics** 子分頁 → 點 **Edit**。
4. 選擇 **Shared Direct（VM with vendor-specific shared graphics）**。
5. *Shared passthrough GPU assignment policy* 留預設 **Spread VMs across GPUs**（分散；本手冊範例兩 VM 落在不同 GPU）。
6. **OK** 套用。

### 7.2 套用設定：重啟服務或重開機

**任選其一**：

#### 方式 A：重啟服務（不需重開機）

SSH 進 ESXi：

```bash
/etc/init.d/xorg stop
nv-hostengine -t
nv-hostengine -d
/etc/init.d/xorg start
```

#### 方式 B：重開機 Host

```bash
esxcli system maintenanceMode set --enable=true
reboot
# 開機後
esxcli system maintenanceMode set --enable=false
```

### 7.3 驗證 Graphics Type 已生效

回到 vCenter → Host → **Configure → Graphics → Host Graphics**，*Default graphics type* 應顯示為 `Shared Direct`。

切到 **Graphics Devices** 子分頁：兩張 GPU 的 *Active Type* 應同為 `Shared Direct`（如果還是 `Shared`，重啟服務或重開機尚未完整生效，請再做一次 §7.2）。

---

## 8. （選用）vGPU 排程策略調整

vGPU 預設使用 **Best Effort（Round-robin）** 排程，適合一般測試。若日後要保證 QoS，可在 ESXi 上調整：

```bash
# Equal Share（依活躍 VM 數量動態分配）
esxcli system module parameters set -m nvidia -p \
  "NVreg_RegistryDwords=RmPVMRL=0x01"

# 改完需重開機 Host
reboot
```

詳細策略對照見 `../doc-NVAIE/06-NVAIE-Advanced-GPU-Config.md` §2。本手冊驗證階段使用預設值即可。

---

## 9. 完成檢查清單

- [ ] `esxcli software vib list | grep nvidia` 顯示 NVAIE VIB
- [ ] `vmkload_mod -l | grep nvidia` 顯示已載入
- [ ] `nvidia-smi` 在 ESXi Host 顯示**兩張** H200 NVL（官方 HBM3e 容量 **141 GB**；`nvidia-smi` 顯示約 `144384 MiB`）
- [ ] vCenter Host → Configure → Graphics → Host Graphics 的 *Default graphics type* = `Shared Direct`
- [ ] Graphics Devices 兩張 GPU 的 *Active Type* = `Shared Direct`
- [ ] Host 已離開 Maintenance Mode，狀態 Connected

→ 全部 ✅ 後，前往 [03-dls-license-server.md](03-dls-license-server.md)。
