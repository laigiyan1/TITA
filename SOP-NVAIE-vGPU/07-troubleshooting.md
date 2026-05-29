# 07 — 疑難排解（Troubleshooting）

> 本章節整理建置過程最常見的失敗症狀。每個小節列：**症狀 → 根因 → 處理步驟**。

---

## 1. ESXi Host：VIB 已裝但 `nvidia-smi` 沒輸出 / 看不到 GPU

### 1.1 症狀
```bash
# ESXi Host SSH
nvidia-smi
# 輸出：command not found 或 No devices were found
```

### 1.2 可能根因 & 處理

| 根因 | 確認方式 | 處理 |
|------|---------|------|
| VIB 沒安裝成功 | `esxcli software vib list | grep nvidia` 沒輸出 | 回 02 §4 重新 `esxcli software vib install`，注意使用 `/vmfs/volumes/` 絕對路徑 |
| 沒重開機 | `vmkload_mod -l | grep nvidia` Loaded = No | 重開機 ESXi（`reboot`） |
| BIOS Above 4G Decoding / MMIO 未啟用 | 進伺服器 BIOS 檢查 | 啟用後重開機 |
| GPU 插槽 PCIe lane 不足 / 沒供電 | iLO / iDRAC 看伺服器健康紀錄 | 換插槽 / 補供電 / 與伺服器廠商確認 |
| 兩張卡只看到一張 | `lspci | grep -i nvidia` 在 ESXi 上看是否兩張 | 若 lspci 已少一張 → 硬體 / BIOS 層次問題；若 lspci 兩張但 nvidia-smi 一張 → 重灌 VIB / 重開機 |

```bash
# 強制重新初始化 NVIDIA host engine（不需重開機，但建議先試）
/etc/init.d/xorg stop
nv-hostengine -t
sleep 3
nv-hostengine -d
/etc/init.d/xorg start
nvidia-smi
```

---

## 2. VM 開機失敗

### 2.1 `Module DevicePowerOn failed`

**根因：** `pciPassthru.64bitMMIOSizeGB` 太小。

**處理：**

1. 關閉 VM。
2. **Edit Settings → VM Options → Advanced → Edit Configuration**。
3. 把 `pciPassthru.64bitMMIOSizeGB` 改為**下一個 2 的次方**：`128 → 256 → 512 → 1024`。
4. 確認 `pciPassthru.use64bitMMIO=TRUE`。
5. 確認 Boot Firmware = **EFI**（非 BIOS）。
6. 確認記憶體 Reservation = 全部 RAM（vGPU 強制要求）。
7. Power On 再試。

> H200 NVL 單卡實務 128 GB 已足夠；若仍失敗，往上加，不要往下試。

### 2.2 `Insufficient graphics resources available on host`

**根因：** Host 的 Graphics Type 還是 `Shared`，不是 `Shared Direct`。

**處理：**

1. vCenter → Host → **Configure → Graphics → Host Graphics → Edit** → 選 `Shared Direct`。
2. 重啟 Xorg / nv-hostengine：
   ```bash
   /etc/init.d/xorg stop
   nv-hostengine -t
   nv-hostengine -d
   /etc/init.d/xorg start
   ```
3. 或重開機 ESXi Host。
4. 回 **Graphics Devices** 確認兩張 H200 NVL 的 *Active Type* = `Shared Direct`。
5. 再 Power On VM。

### 2.3 `Unable to retrieve the host GPU information`

**根因：** VIB 未生效（裝完忘了 reboot）或核心模組沒載入。

**處理：** 回 02 §6 三項驗證；若 `vmkload_mod -l | grep nvidia` 顯示 Loaded = No → 重開機 ESXi。

### 2.4 `The specified vGPU type ... is not supported with other ...`

**根因：** 同一張實體 GPU 上已經有不同 profile 的 vGPU。預設 homogeneous 模式不允許混 profile。

**處理：** 兩個 VM 用相同 profile，或讓兩 VM 落在不同 GPU（透過 `pciPassthru0.cfg.gpu-pci-id` 指定）。

---

## 3. License Status 一直是 Unlicensed（最常見）

### 3.1 症狀
```bash
nvidia-smi -q | grep "License Status"
# License Status : Unlicensed (Restricted)
# 或 Unlicensed (Unrestricted)
```

### 3.2 排查順序

```bash
# 1. 服務有沒有起來
sudo systemctl status nvidia-gridd --no-pager

# 2. 看 log 找錯誤
sudo journalctl -u nvidia-gridd --no-pager | tail -50
```

| Log 訊息 | 根因 | 處理 |
|---------|------|------|
| `No token found in /etc/nvidia/ClientConfigToken` | Token 沒放對位置 | 確認 `.tok` 在 `/etc/nvidia/ClientConfigToken/`，權限 744 |
| `Failed to connect to license server` | 網路不通 / DNS 失敗 | `curl -k https://<dls-ip>` 測試；檢查 DLS 防火牆、Guest 路由 |
| `Token signature verification failed` | Token 損毀或過期 | 重新到 DLS 產生新 Token |
| `License pool exhausted` | DLS 沒有足夠 feature | 在 Licensing Portal 增加 feature 數量，或回收沒在用的 lease |
| `Invalid system identity` | Token 跟 DLS Instance 不匹配 | 重新從**正確的 DLS**下載 Token |

### 3.3 套用 Token 後手動觸發

```bash
# 一定要 restart，不能只 reload
sudo systemctl restart nvidia-gridd

# 再驗證
sudo journalctl -u nvidia-gridd --since "1 minute ago"
nvidia-smi -q | grep -A 3 "License Status"
```

### 3.4 確認網路通到 DLS

```bash
# Guest 內
curl -kv https://<dls-ip>/  2>&1 | head -20
```

✅ 預期：HTTPS handshake 成功，回應為 DLS Web UI 的 HTML。

❌ 失敗：檢查 Guest 預設路由、DLS 防火牆 (TCP 443)、中間網路設備 ACL。

### 3.5 DLS 端排查

到 DLS Web UI：

- **AUDIT LOG** 看是否有來自此 VM 的請求紀錄
- **LICENSE SERVERS → installed licenses** 確認還有可用 feature

---

## 4. Guest 內 `nvidia-smi` 找不到 GPU

### 4.1 症狀

```bash
nvidia-smi
# NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
```

### 4.2 排查

```bash
# 1. PCI 上看得到 vGPU 嗎？
lspci | grep -i nvidia
# 應該有 NVIDIA Corporation Device ...

# 2. nouveau 是不是又跑出來？
lsmod | grep nouveau
# 應該無輸出；有輸出代表 blacklist 沒生效

# 3. NVIDIA 模組有載入嗎？
lsmod | grep nvidia
# 應該看到 nvidia / nvidia_drm / nvidia_modeset / nvidia_uvm

# 4. driver 版本確認
modinfo nvidia | grep version
```

### 4.3 處理

| 狀況 | 處理 |
|------|------|
| `lspci` 看不到 NVIDIA → vCenter 沒加 PCI Device | 關 VM，回 04 §4.4 加 vGPU |
| nouveau 還在 | 重新做 05 §1.5（blacklist + update-initramfs + reboot） |
| nvidia 模組沒載入 | `sudo modprobe nvidia` 看錯誤；通常是 kernel 升級 → Driver 沒重編 |
| Kernel 升級導致 Driver 失效 | 若安裝時有選 DKMS → 通常會自動 rebuild；沒選 DKMS → 重新跑 `.run` |

---

## 5. Grid Driver 安裝失敗

> 章節 05 主用 `.deb` 安裝法（對齊 [`GPU-passthrough-sop.md`](GPU-passthrough-sop.md) 風格），以下排查依序處理。

### 5.1 `dpkg -i` 報缺依賴

```
dpkg: dependency problems prevent configuration of nvidia-linux-grid-595:
 nvidia-linux-grid-595 depends on ...
```

**處理：**

```bash
root@vgpu-test-01:/home/ubuntu# apt-get install -f -y
root@vgpu-test-01:/home/ubuntu# dpkg -l | grep -i nvidia    # 確認 install 後狀態為 ii
```

### 5.2 安裝完 reboot 後 `nvidia-smi` 報錯

```
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
Make sure that the latest NVIDIA driver is installed and running.
```

**常見根因 & 處理：**

| 根因 | 確認 | 處理 |
|------|------|------|
| 裝錯成 Data Center Driver（非 -grid） | `dpkg -l | grep -i nvidia`；若看到 `cuda-drivers` / `nvidia-driver-XXX-server` 而非 `nvidia-linux-grid-XXX` → 裝錯 | 完全移除 DC driver，重裝 `nvidia-linux-grid-*.deb` |
| nouveau 還在 | `lsmod | grep nouveau` 有輸出 | 重做 05 §3 blacklist + `update-initramfs -u` + reboot |
| kernel 升級導致模組失效 | `uname -r` 與 `dkms status` 不一致 | `apt install -y linux-headers-$(uname -r)` 後重新 `dpkg --configure -a` / 重裝 deb |
| Secure Boot 啟用未簽署 | `mokutil --sb-state` 顯示 enabled | 章節 01 §2 已建議停用，或依 MOK 流程簽署模組 |

### 5.3 完全移除舊 Driver（換版或從 DC driver 切換 Grid driver 必做）

```bash
root@vgpu-test-01:/home/ubuntu# apt-get -y remove --purge "nvidia-*" "cuda-*" "libnvidia-*"
root@vgpu-test-01:/home/ubuntu# apt-get -y autoremove
root@vgpu-test-01:/home/ubuntu# reboot
# 重開後再依 05 §4 重新 dpkg -i 對應 -grid deb
```

### 5.4 備援方案：用 `.run` 安裝

若 `.deb` 因環境因素無法成功，可改用 NGC ZIP 內的 `NVIDIA-Linux-x86_64-595.<patch>-grid.run`：

```bash
root@vgpu-test-01:/home/ubuntu# apt-get install -y build-essential dkms linux-headers-$(uname -r)
root@vgpu-test-01:/home/ubuntu# chmod +x NVIDIA-Linux-x86_64-595.<patch>-grid.run
root@vgpu-test-01:/home/ubuntu# ./NVIDIA-Linux-x86_64-595.<patch>-grid.run
```

安裝精靈詢問 DKMS 時選 **Yes**（kernel 升級會自動 rebuild）。`.run` 與 `.deb` **不可同時並存**，請擇一。

### 5.5 Secure Boot 啟用導致模組無法載入

本手冊預設不啟用 Secure Boot。若一定要開：

- `.deb` 安裝路徑：deb 安裝後需手動以 MOK 簽署 `nvidia*.ko`（複雜，不建議）
- `.run` 安裝路徑：安裝精靈會引導 MOK 簽署（給密碼 → reboot → 在 MokManager 註冊）
- 完成後 `lsmod | grep nvidia` 看到 nvidia 模組才算成功

---

## 6. 兩個 VM 落到同一張 GPU，導致一個無法開機

### 6.1 症狀

VM #1 開機正常，VM #2 開機報錯 `not enough resources` 或 `vGPU profile mismatch`。

### 6.2 根因

`Shared passthrough GPU assignment policy` 設為 `Consolidate VMs onto a single GPU`（預設應為 `Spread VMs across GPUs`），導致兩個 VM 想擠到同一張 GPU 但 profile 配置不容。

### 6.3 處理

**方式 A：改 host policy**

vCenter → Host → **Configure → Graphics → Host Graphics → Edit** → *Shared passthrough GPU assignment policy* 改為 `Spread VMs across GPUs`。

**方式 B：強制指定 GPU**

在每個 VM 的進階參數加入：

```
# vgpu-test-01 → GPU #0
pciPassthru0.cfg.gpu-pci-id = "0000:31:00.0"

# vgpu-test-02 → GPU #1
pciPassthru0.cfg.gpu-pci-id = "0000:CA:00.0"
```

> BDF 從 ESXi `lspci | grep -i nvidia` 取得，務必符合你環境的實際位址。

---

## 7. 系統重開機後 vGPU 無法重新取得授權

### 7.1 症狀

VM 重開機後 `nvidia-smi -q | grep License Status` 又變 `Unlicensed`。

### 7.2 排查

```bash
sudo systemctl is-enabled nvidia-gridd
# 應該是 enabled；若是 disabled：
sudo systemctl enable nvidia-gridd

sudo systemctl status nvidia-gridd
```

如果 nvidia-gridd 已 enabled 但開機沒成功 acquire license：

```bash
sudo journalctl -u nvidia-gridd -b --no-pager | head -50
```

通常根因是 **DLS 還沒起來時 VM 已嘗試 acquire license**，service 重啟一次即可：

```bash
sudo systemctl restart nvidia-gridd
```

若要永久解決：將 nvidia-gridd 設定為依賴 network-online 或加 systemd timer 重試（不在本手冊範圍）。

---

## 8. 還有問題？

| 來源 | 用途 |
|------|------|
| `../doc-NVAIE/07-NVAIE-Deployment-Checklist.md` | 完整部署 checklist，逐項對照 |
| NVIDIA VMware 部署文件 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/index.html |
| NVAIE Support Forum | https://forums.developer.nvidia.com/c/accelerated-computing/cuda/206 |
| NVIDIA Enterprise Support Portal | https://enterprise.nvidia.com |
| VMware KB 文章 | https://knowledge.broadcom.com/ |
