# NVAIE 建立第一台 AI VM

> 來源：https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/first-vm.html  
> 整理日期：2026-05-20

---

## 1. 兩種 GPU 存取模式比較

| 特性 | GPU Passthrough (Direct Path I/O) | vGPU |
|------|-----------------------------------|------|
| 每 VM GPU 數 | 1 張完整 GPU（HGX 上可多 GPU） | 1 張 GPU 的分片（HGX on vSphere 支援 1/2/4/8 GPU per VM） |
| VM 數量 | 1:1（1 GPU = 1 VM） | 1:N（1 GPU = 多 VM） |
| 性能 | 接近 Bare Metal | 略低，可控 |
| Live Migration | ❌ 不支援 | ✅ 支援（vMotion） |
| License Server 需求 | ❌ 不需要 NLS Client Leasing | ✅ 需要 License Server（CLS/DLS） |
| NVAIE 訂閱授權 | ✅ 仍需要（per-GPU） | ✅ 需要（per-GPU） |
| 驅動類型 | Data Center Driver | Grid Guest Driver |
| MMIO 設定 | ✅ 必需（EFI + 64-bit MMIO） | ✅ 必需（EFI + 64-bit MMIO，C-Series） |

---

## 2. 建立 VM（透過 vSphere Web Client）

### 2.1 基本建立流程
1. 登入 vCenter Server
2. 右鍵選取 Cluster 或 Host → **New Virtual Machine**
3. 選擇 **Create a new virtual machine**
4. 設定 VM 名稱與存放位置
5. 選擇計算資源（含 NVIDIA GPU 的 ESXi Host）
6. 選擇 Datastore
7. 設定相容性（符合 ESXi 版本）
8. 選擇 Guest OS：**Ubuntu Linux 64-bit**

### 2.2 推薦 VM 規格

| 資源 | 建議值 |
|------|--------|
| vCPU | 16（單 Socket） |
| RAM | 64 GB |
| 磁碟 | 150 GB（Thin Provisioned） |
| 網路介面卡 | VMXNET3 |

---

## 3. GPU Passthrough 模式設定

### 3.1 在 vSphere 中啟用 PCI Passthrough
1. 選取 ESXi Host → **Configure** → **Hardware** → **PCI Devices**
2. 找到 NVIDIA GPU → 點選 **Toggle Passthrough**
3. 重開機 ESXi Host

### 3.2 將 GPU 加入 VM
1. 關閉 VM
2. 編輯 VM 設定 → **Add New Device** → **PCI Device**
3. 選取已設定 Passthrough 的 NVIDIA GPU
4. 儲存設定

### 3.3 必要的 VM 進階設定（MMIO）

先將 VM Boot Firmware 改為 **EFI**（VM Options → Boot Options → Firmware → EFI）

再加入以下 Advanced Configuration Parameters。以下數值來自 NVIDIA 官方文件 (`first-vm.html`) 的 MMIO 表格：

| GPU 型號 | pciPassthru.64bitMMIOSizeGB | pciPassthru.use64bitMMIO |
|----------|------------------------------|--------------------------|
| A100 80GB（全系列） | 256 | TRUE |
| A100 40GB（全系列） | 128 | TRUE |
| A40 | 128 | TRUE |
| A10 / A30 | 64 | TRUE |
| Tesla P100（全系列） | 64 | TRUE |
| RTX A5000 / A5500 | 64 | TRUE |
| RTX A6000 | 128 | TRUE |

> ⚠️ **H100 / H200 MMIO 值**：NVIDIA 官方 `first-vm.html` 的 MMIO 表格目前未列出 H100/H200。若需設定，請以 A100 80GB（256 GB）作為估算參考，但應視為**待驗證值**，部署前務必查閱最新官方文件或聯繫 NVIDIA 支援確認。  
> 來源：https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/first-vm.html

> **多 GPU Passthrough：** 每個 GPU 都需要對應的 `pciPassthru<N>.cfg` 設定，N 從 0 開始遞增。

---

## 4. vGPU 模式設定

### 4.1 VM 設定需求
- Boot Firmware：**EFI**（C-Series vGPU 必須）
- 啟用 **64-bit MMIO**（C-Series vGPU 需要，與 Passthrough 相同機制）
- MMIO 大小依 GPU 型號設定（參考 §3.3 表格；H100/H200 請查官方文件確認）

### 4.2 指派 vGPU Profile
1. 關閉 VM
2. 編輯 VM 設定 → **Add New Device** → **PCI Device**
3. 選擇 vGPU Profile（必須是 **C-series**，例如 `H100-20C`）
4. 開啟 VM

### 4.3 H100 / H200 vGPU Profile（C-series）正式名稱

Profile 名稱依 GPU 型號（PCIe vs SXM5）及容量而不同，**不要使用 `GRID H100-*` 這類舊式前綴寫法**。

#### H100 PCIe 80GB

| Profile 名稱 | 記憶體 | 最大 vGPU 數/實體 GPU |
|-------------|--------|----------------------|
| H100-80 | 81 GB | 1 |
| H100-40C | 40 GB | 2 |
| H100-20C | 20 GB | 4 |
| H100-16C | 16 GB | 6 |
| H100-10C | 10 GB | 8 |
| H100-8C | 8 GB | 10 |
| H100-5C | 5 GB | 16 |
| H100-4C | 4 GB | 20 |

#### H100 SXM5 80GB（HGX）

| Profile 名稱 | 記憶體 | 最大 vGPU 數/實體 GPU |
|-------------|--------|----------------------|
| H100XM-80C | 81 GB | 1 |
| H100XM-40C | 40 GB | 2 |
| H100XM-20C | 20 GB | 4 |
| H100XM-16C | 16 GB | 6 |
| H100XM-10C | 10 GB | 8 |
| H100XM-8C | 8 GB | 10 |
| H100XM-5C | 5 GB | 16 |
| H100XM-4C | 4 GB | 20 |

#### H200 PCIe 141GB（即 H200 NVL）

NVIDIA 官方表格標題為 `NVIDIA H200 PCIe 141GB (H200 NVL)`，以下 `H200-*` profiles 即為此型號所用。

| Profile 名稱 | 記憶體 | 最大 vGPU 數/實體 GPU |
|-------------|--------|----------------------|
| H200-141C | 144 GB | 1 |
| H200-70C | 71 GB | 2 |
| H200-35C | 35 GB | 4 |
| H200-28C | 28 GB | 5 |
| H200-17C | 17 GB | 8 |
| H200-14C | 14 GB | 10 |
| H200-8C | 8 GB | 16 |
| H200-7C | 7 GB | 20 |
| H200-4C | 4 GB | 32 |

#### H200 SXM5 141GB

H200 SXM5 使用 `H200X-*` 前綴，與 PCIe/NVL 版本不同。部分範例：

| Profile 名稱（範例） | 記憶體 |
|---------------------|--------|
| H200X-141C | 144 GB |
| H200X-70C | 71 GB |
| H200X-35C | 35 GB |

> 完整清單請查閱官方 vGPU Types Reference。部署前務必依實際 GPU form factor（PCIe/NVL 用 `H200-*`；SXM5 用 `H200X-*`）選擇正確 Profile。

> 來源：https://docs.nvidia.com/ai-enterprise/release-7/7.5/infra-software/vgpu/reference/hopper.html  
> MIG-backed vGPU Profile 格式示例（H100 PCIe）：`H100-4-40C`、`H100-1-10C`（中間第一個數字代表 MIG GI 大小）

---

## 5. Ubuntu Guest OS 安裝

### 5.1 安裝步驟
1. 掛載 Ubuntu Server 22.04 LTS ISO 至 VM CD/DVD Drive
2. 開機並選擇 Install Ubuntu Server
3. 設定語言、鍵盤、網路（建議靜態 IP）
4. 磁碟格式化（使用整個磁碟）
5. 安裝 OpenSSH Server（勾選）
6. 建立管理者帳號
7. 重開機後移除 ISO

### 5.2 安裝後基本設定
```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install build-essential -y
```

---

## 6. GPU Passthrough 模式：安裝 Data Center Driver

### 6.1 下載 Driver
- 前往 NVIDIA Driver 下載頁面
- 選擇：Data Center/Tesla → 您的 GPU 架構 → Linux 64-bit

### 6.2 安裝步驟
```bash
# 安裝編譯工具
sudo apt-get install build-essential -y

# 加入執行權限
sudo chmod +x NVIDIA-Linux-x86_64-<version>.run

# 執行安裝
sudo sh ./NVIDIA-Linux-x86_64-<version>.run

# 重開機
sudo reboot

# 驗證
nvidia-smi
```

---

## 7. vGPU 模式：安裝 Grid Guest Driver

### 7.1 下載 Driver（從 NGC）
```bash
# 設定 NGC CLI
ngc config set

# 下載 vGPU Guest Driver（資源路徑格式：nvidia/vgpu/vgpu-guest-driver-<N>:<version>）
# 其中 <N> 為 vGPU 主版號（例如 19、20），<version> 為完整版本號
# 範例（實際版本號請至 NGC 確認）：
ngc registry resource download-version "nvidia/vgpu/vgpu-guest-driver-19:<version>"
```

> **注意**：NGC 資源路徑為 `nvidia/vgpu/vgpu-guest-driver-<N>`，而非 `nvidia/driver/vgpu-guest-driver-linux`。NVAIE 7.5 對應 vGPU 19.x，NVAIE 8.1 對應 vGPU 20.x。實際版本號請登入 NGC 查詢最新版：https://ngc.nvidia.com

### 7.2 安裝步驟（Ubuntu）
```bash
# 安裝編譯工具
sudo apt-get install build-essential -y

# 執行安裝（Grid Driver 有 -grid 後綴）
sudo sh ./NVIDIA-Linux-x86_64-<version>-grid.run

# 重開機
sudo reboot

# 驗證
nvidia-smi
```

---

## 8. 套用 vGPU 授權（vGPU 模式）

```bash
# 建立 Token 目錄
sudo mkdir -p /etc/nvidia/ClientConfigToken/

# 複製從 DLS 產生的 Token 檔案
sudo cp client_configuration_token_*.tok /etc/nvidia/ClientConfigToken/

# 設定 Token 檔案權限
sudo chmod 744 /etc/nvidia/ClientConfigToken/*.tok

# 重啟授權服務
sudo systemctl restart nvidia-gridd

# 檢查授權狀態
nvidia-smi -q | grep -A 10 "vGPU Software Licensed Product"
```

---

## 9. 驗證 GPU 在 VM 中可用

```bash
# 確認 GPU 可見
nvidia-smi

# 確認 CUDA 可用（若已安裝 CUDA）
nvidia-smi -L

# 測試 GPU 運算
docker run --rm --gpus all nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi
```
