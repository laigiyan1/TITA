# 06 — 雙 VM 與 Host 端完整驗證

> 對應流程：階段 7
> 本章節提供「全建置完成」後可逐項勾選的驗證指令集，橫跨：
> 1. ESXi Host 端
> 2. DLS 端
> 3. 兩個 vGPU VM Guest 端
>
> 全部 ✅ 才算手冊目標達成。

---

## 1. ESXi Host 端驗證

> SSH 進 ESXi 8.0U3 主機（`ssh root@<esxi-ip>`）執行以下指令。

### 1.1 VIB 與核心模組

```bash
# VIB 仍掛載
esxcli software vib list | grep -i nvidia

# 核心模組已載入
vmkload_mod -l | grep nvidia
```

✅ 預期：兩個指令都有輸出，無錯誤訊息。

### 1.2 兩張實體 H200 NVL 都健康

```bash
nvidia-smi
```

✅ 預期：
- 兩張 `NVIDIA H200 NVL` 顯示，官方 HBM3e 容量 **141 GB**（`nvidia-smi` Memory-Usage 欄會顯示約 `144384 MiB`）
- 溫度正常（< 80 °C 待機）
- 功耗有合理值（待機約 60–80 W）
- **Memory-Usage 欄位顯示有部分使用**（因為兩個 vGPU 已切出，占走部分框緩衝；每個 VM 對應一個 `nvidia_h200-70c` profile，官方 profile 容量 **70 GB**，所以每張卡會用掉約 70 GB framebuffer；`nvidia-smi` MiB 顯示值會落在 72704 MiB 上下，屬正常換算差異）

### 1.3 兩個 vGPU 實例已對外分派

```bash
nvidia-smi vgpu
```

✅ 預期輸出（樣式示意）：

```
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.71.05            Driver Version: 595.71.05                             |
|---------------------------------+------------------------------+----------------------+
| GPU  Name                       | Bus-Id                       | GPU-Util             |
|      vGPU ID     Name           | VM ID     VM Name            | vGPU-Util            |
|=================================+==============================+======================|
|   0  NVIDIA H200 NVL            | 00000000:31:00.0             |   0%                 |
|      3251634xxxx NVIDIA H200-70C| 5xxxxxx    vgpu-test-01      |      0%              |
+---------------------------------+------------------------------+----------------------+
|   1  NVIDIA H200 NVL            | 00000000:CA:00.0             |   0%                 |
|      3251634yyyy NVIDIA H200-70C| 6xxxxxx    vgpu-test-02      |      0%              |
+---------------------------------+------------------------------+----------------------+
```

✅ 必須看到：
- 兩個 vGPU 實例（vGPU ID 不同）
- vGPU type 為 `NVIDIA H200-70C`
- VM Name 對應 `vgpu-test-01` 與 `vgpu-test-02`

### 1.4 各 vGPU 實例細節

```bash
nvidia-smi vgpu -q
```

✅ 預期：每個 vGPU instance 區塊都顯示
- License Status: `Licensed`（Token 已套用）
- Frame Buffer 使用量
- 對應的 VM UUID

```bash
nvidia-smi vgpu -u
```

✅ 預期：顯示每個 vGPU 的 SM / encoder / decoder utilization（待機通常都是 0%）。

### 1.5 Graphics Type 仍為 Shared Direct

```bash
# 從 ESXi 端不易直接查 Graphics Type，請至 vCenter Web Client 確認：
# Host → Configure → Hardware → Graphics → Host Graphics
# Default graphics type: Shared Direct ✅
# Graphics Devices 分頁：兩張 H200 NVL 的 Active Type 都是 Shared Direct ✅
```

---

## 2. DLS 端驗證

登入 DLS Web UI（`https://<dls-ip>`）→ Application User：

| 檢查項目 | 預期 |
|---------|------|
| **DASHBOARD → License Server `nvaie-h200nvl-pool`** | 顯示 `2 LEASED`（兩個 VM 各租用 1 個 NVAIE feature） |
| **LICENSE SERVERS → nvaie-h200nvl-pool → CONNECTED CLIENTS** | 列出兩筆 client，名稱 / UUID 對應 `vgpu-test-01` 與 `vgpu-test-02`，狀態 `ACTIVE` |
| Service Instance 健康狀態 | `OK` / Online |
| Token 剩餘有效期 | > 預期使用時間 |

> 若顯示 `0 LEASED` 或只有 1 個 → 表示某個 VM 的 nvidia-gridd 沒成功從 DLS 取得 license，回 [07-troubleshooting.md](07-troubleshooting.md) §3。

---

## 3. 兩個 vGPU VM Guest 端驗證

> 對 `vgpu-test-01` 與 `vgpu-test-02` **各執行一次**。

### 3.1 vGPU 可見

```bash
nvidia-smi
```

✅ 預期：
- Name = `NVIDIA H200-70C`
- Memory Total ≈ 72704 MiB（官方 profile 容量 = **70 GB**；MiB 顯示為實際暴露的 Frame Buffer，與 70 GB 略有換算差異屬正常）
- Power / Temp = `N/A`（vGPU Guest 視角下正常）
- Bus-Id 為 vGPU 在 PCI bus 中的虛擬位址（例 `00000000:02:01.0`）

### 3.2 授權正常

```bash
nvidia-smi -q | grep -A 5 "vGPU Software Licensed"
```

✅ 預期：

```
    vGPU Software Licensed Product
        Product Name              : NVIDIA AI Enterprise
        License Status            : Licensed (Expiry: YYYY-MM-DD HH:MM:SS)
```

### 3.3 nvidia-gridd 服務健康

```bash
sudo systemctl status nvidia-gridd --no-pager | head -10
sudo journalctl -u nvidia-gridd --no-pager | tail -20
```

✅ 預期：service `active (running)`，log 末尾有 `License acquired successfully.` 字樣。

### 3.4 結構化查詢（適合自動化）

```bash
nvidia-smi --query-gpu=name,memory.total,memory.used,utilization.gpu,driver_version --format=csv,noheader
```

✅ 預期一行類似：

```
NVIDIA H200-70C, 72704 MiB, 0 MiB, 0 %, 595.71.05
```

### 3.5 持續監控指令（可用於壓測時觀察）

```bash
nvidia-smi dmon -s u -d 1
```

啟動後每秒輸出一行 vGPU 使用率，按 Ctrl+C 結束。

### 3.6 容器驗證（章節 05 §7–§9 已安裝 Docker + Container Toolkit）

```bash
# 對齊章節 05 §9.1 / GPU-passthrough-sop §9 的指令格式
root@vgpu-test-01:~# docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

✅ 預期：容器內輸出與 §3.1 一致，顯示 `NVIDIA H200-70C`（官方 profile 容量 **70 GB**；`nvidia-smi` 可能顯示約 72704 MiB）。

進階（拉 NVAIE CUDA 映像，前提是章節 05 §10.7 已 `docker login nvcr.io`）：

```bash
root@vgpu-test-01:~# docker run --rm --runtime=nvidia --gpus all nvcr.io/nvidia/cuda:12.8.1-base-ubuntu24.04 nvidia-smi
```

> Container Toolkit 與 Docker 在 vGPU 上完全相容；vGPU **不支援 DCGM** 完整堆疊（見 `../doc-NVAIE/08-NVAIE-GPU-Monitoring.md` §4），故監控以 `nvidia-smi` 為主。

---

## 4. 最終目標達成檢查表

打勾即代表本手冊目標完成：

### 基礎設施
- [ ] **ESXi 8.0U3 P06 或更新的 8.0 update**（依 NVIDIA vGPU 20.0/20.1 VMware vSphere Release Notes 要求）+ vCenter 8.0U3 正常運作，Host 已 Add 進 vCenter
- [ ] VMware 授權 edition 為 vSphere Foundation 或 Enterprise Plus
- [ ] H200 NVL × 2 在 ESXi `nvidia-smi` 可見
- [ ] NVAIE 8.1 VIB 已安裝
- [ ] Graphics Type = `Shared Direct`

### 授權
- [ ] DLS VM 上線，與 Licensing Portal 已 bind
- [ ] License Pool 設定完成（≥ 2 個 NVAIE features）
- [ ] DLS Web UI 顯示 `2 LEASED` 給 `vgpu-test-01` 與 `vgpu-test-02`

### VM #1（`vgpu-test-01`）
- [ ] EFI Firmware；MMIO `use64bitMMIO=TRUE` + `64bitMMIOSizeGB=128`
- [ ] 已掛 `nvidia_h200-70c` vGPU
- [ ] Ubuntu 24.04 LTS 已安裝
- [ ] Grid Driver 595.71.05 已安裝
- [ ] `nvidia-smi` 顯示 `NVIDIA H200-70C`，官方 profile 容量 **70 GB**（`nvidia-smi` 可能顯示約 72704 MiB，屬換算差異）
- [ ] License Status = `Licensed`

### VM #2（`vgpu-test-02`）
- [ ] 與 VM #1 同樣設定全部 ✅

### Host 端 vGPU 視角
- [ ] `nvidia-smi vgpu` 列出 2 個 vGPU 實例
- [ ] 兩個 vGPU 對應到正確的 VM Name
- [ ] 兩個 vGPU 對應到不同實體 GPU（GPU 0 / GPU 1）— 若採章節 00 §5 預設方案

🎉 全部 ✅ → 手冊目標達成：兩個 Ubuntu 24.04 VM 各自透過 `nvidia_h200-70c` profile 取得 vGPU 資源，並通過完整 vGPU 狀態驗證。

---

## 5. 後續延伸（選用）

若你想進一步將此測試環境用於實際 AI workload，可參考 doc-NVAIE 內既有資料：

| 主題 | 對應文件 |
|------|---------|
| Docker + NVIDIA Container Toolkit | `../doc-NVAIE/05-NVAIE-Docker-Container-Toolkit.md` |
| vGPU 環境 GPU 監控（nvidia-smi 為主，DCGM 不支援） | `../doc-NVAIE/08-NVAIE-GPU-Monitoring.md` §4 |
| 進階 vGPU 排程策略 / 多 GPU VM / MIG | `../doc-NVAIE/06-NVAIE-Advanced-GPU-Config.md` |

---

## 6. 出問題了？

任何一項驗證失敗，請翻 [07-troubleshooting.md](07-troubleshooting.md) 對應章節：

| 失敗症狀 | 跳轉到 |
|---------|--------|
| ESXi Host `nvidia-smi` 沒輸出 / 看不到 GPU | 07 §1 |
| VM 開機失敗（`Module DevicePowerOn failed`） | 07 §2 |
| VM 開機失敗（`Insufficient graphics resources`） | 07 §2 |
| Guest 內 `nvidia-smi` 找不到 GPU | 07 §4 |
| Guest 內 License Status 一直是 Unlicensed | 07 §3 |
| nvidia-gridd 服務啟動失敗 | 07 §3 |
| Driver 安裝時 kernel module 編譯失敗 | 07 §5 |
