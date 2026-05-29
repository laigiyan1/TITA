# NVAIE Host Software (VIB) 安裝指南

> 來源：https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/vgpu.html  
> 整理日期：2026-05-20

---

## 1. 概覽

NVAIE Host Software 以 **VIB (vSphere Installation Bundle)** 形式安裝在 ESXi Host 上，提供：
- vGPU 功能（多 VM 共享 GPU）
- vSGA（Virtual Shared Graphics Acceleration）
- 裝置管理與 GPU 監控

---

## 2. 安裝前準備

### 2.1 開啟 ESXi Shell 和 SSH
（在安裝 ESXi 後需要先開啟）
- 在 vSphere Web Client：Host → Configure → System → Security Profile
- 啟用 ESXi Shell 和 SSH

### 2.2 將 VIB 上傳到 Datastore
1. 在 vSphere Web Client 選取 ESXi Host
2. 點選 **Datastores** 標籤
3. 右鍵點選 Datastore → **Browse Files**
4. 建立新資料夾命名為 `VIB`
5. 上傳 `NVIDIA-AIE_ESXi_<version>.vib` 到此資料夾

---

## 3. 安裝步驟

### 3.1 進入維護模式
```bash
# 透過 SSH 登入 ESXi Host，執行：
esxcli system maintenanceMode set --enable=true

# 或在 vSphere Web Client：右鍵 Host → Enter Maintenance Mode
```

> **注意**：進入維護模式前，必須先關閉所有 VM。

### 3.2 安裝 VIB
```bash
# 安裝指令（使用 /vmfs/volumes/ 絕對路徑）
esxcli software vib install -v /vmfs/volumes/<datastore-name>/VIB/NVIDIA-AIE_ESXi_<version>.vib

# 範例（僅供格式示意，570.x 為 Infra 6.x 驅動版號，已 EOL 2026/03）：
esxcli software vib install -v /vmfs/volumes/datastore1/VIB/NVIDIA-AIE_ESXi_570.216.04-1OEM.800.1.0.20613240.x86_64.vib
```

> **重要**：必須使用 `/vmfs/volumes/` 開頭的絕對路徑，不能用 `ds:///` 格式。  
> **版本說明**：上述範例中的 `570.216.04` 屬於 Infra 6.x（已 EOL）。實際部署請從 NGC 下載對應 NVAIE 版本的 VIB：Infra 8.1 對應 R595 驅動（595.x），Infra 7.5 LTSB 對應 R580 驅動（580.x）。請以下載的實際檔名為準。（來源：[NVAIE Active Releases](https://docs.nvidia.com/ai-enterprise/index.html)）

### 3.3 重新啟動 ESXi Host
```bash
reboot
```
即使安裝輸出顯示 "Reboot Required: false"，**仍必須重開機**以載入核心模組和初始化 Xorg。

---

## 4. 驗證安裝

重開機後，透過 SSH 登入 ESXi Host 執行：

```bash
# 驗證 NVIDIA 核心模組已載入
vmkload_mod -l | grep nvidia

# 驗證 GPU 通訊正常（顯示所有實體 GPU）
nvidia-smi
```

`nvidia-smi` 應顯示伺服器中所有 GPU 的資訊（型號、記憶體、驅動版本等）。

---

## 5. 設定 Graphics Type（vGPU 必要步驟）

**預設值為 Shared（只支援 vSGA），若要使用 vGPU 必須改為 Shared Direct。**

### 透過 vSphere Web Client 設定：
1. 登入 vCenter Server
2. 選取 ESXi Host → **Configure** 標籤
3. 選擇 **Graphics** → **Host Graphics**
4. 點選 **Edit**
5. 選取 **Shared Direct**
6. 套用設定

### 套用後：可選擇重開機，或重啟服務：
```bash
# 停止 Xorg
/etc/init.d/xorg stop

# 停止 nv-hostengine
nv-hostengine -t

# 啟動 nv-hostengine
nv-hostengine -d

# 啟動 Xorg
/etc/init.d/xorg start
```

> **警告**：若未設定為 Shared Direct，vGPU VM 啟動時會出現 **"insufficient graphics resources"** 錯誤。

---

## 6. 更新 VIB

```bash
# 更新現有 VIB（不需先 remove）
esxcli software vib update -v /vmfs/volumes/<datastore>/VIB/NVIDIA-AIE_ESXi_<new-version>.vib

# 更新後重開機
reboot
```

---

## 7. 移除 VIB

```bash
# 查詢已安裝的 NVIDIA VIB 名稱
esxcli software vib list | grep -i nvidia

# 移除 VIB
esxcli software vib remove -n NVIDIA-AIE_ESXi_<version> --maintenance-mode

# 重開機
reboot
```

---

## 8. 常用指令速查

| 任務 | 指令 |
|------|------|
| 進入維護模式 | `esxcli system maintenanceMode set --enable=true` |
| 離開維護模式 | `esxcli system maintenanceMode set --enable=false` |
| 持續監控 GPU | `nvidia-smi -l` |
| 列出已安裝 VIB | `esxcli software vib list | grep -i nvidia` |
| 查看 GPU 拓撲 | `nvidia-smi topo -m` |

---

## 9. 注意事項

- vGPU 啟用的 VM，在 vSphere Web Client 無法查看 VM Console 輸出
- 需要 ESXi Shell 或 SSH 才能執行命令列安裝
- 文件中的驅動版本號為示意用，實際版本以 NGC 下載的最新版為準
