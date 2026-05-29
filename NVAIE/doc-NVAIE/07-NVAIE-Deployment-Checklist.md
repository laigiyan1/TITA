# NVAIE VMware 部署檢查清單

> 整理日期：2026-05-20  
> 目標環境：H100/H200 GPU、VMware vSphere 8.x、Ubuntu 22.04、Docker

---

## 情境一：VMware VM + Direct Path I/O GPU Passthrough

### 階段 1：環境準備

- [ ] 確認伺服器為 NVIDIA-Certified System（查詢：https://www.nvidia.com/en-us/data-center/products/certified-systems/）
- [ ] 確認 GPU 型號與 NVAIE 相容（H100/H200 ✅）
- [ ] 確認 VMware vSphere 版本（建議 8.0+）
- [ ] 取得所需軟體授權與 NGC 帳號
- [ ] 規劃網路架構（Management / VM Data / Storage 分離）

### 階段 2：伺服器 BIOS 設定

- [ ] 啟用 Hyperthreading
- [ ] 設定 Power Mode 為 High Performance
- [ ] 啟用 VT-d / IOMMU（DirectPath I/O Passthrough 必要；來源：[VMware DirectPath I/O Requirements](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-resource-management/GUID-F64E7B0C-A2CD-4F85-8440-AD1C0A3BDCAE.html)）
- [ ] 啟用 SR-IOV（若使用 A100）
- [ ] 啟用 Memory Mapped I/O above 4GB
- [ ] 設定 CPU Performance 為 Enterprise / High Throughput

### 階段 3：安裝 VMware ESXi

- [ ] 下載 ESXi ISO（從 Broadcom）
- [ ] 建立 USB 開機媒體
- [ ] 安裝 ESXi（設定 root 密碼、IP、DNS、Hostname）
- [ ] 啟用 ESXi Shell 和 SSH

### 階段 4：安裝 VMware vCenter Server（選用）

- [ ] 部署 vCenter Server Appliance（VCSA）
- [ ] 建立 Datacenter 和 Cluster
- [ ] 加入 ESXi Host 至 Cluster

### 階段 5：設定 GPU Passthrough

- [ ] 在 vSphere 中選取 ESXi Host → Configure → Hardware → PCI Devices
- [ ] 找到 NVIDIA GPU → 點選 Toggle Passthrough
- [ ] 重開機 ESXi Host
- [ ] 確認 GPU 顯示為 Passthrough 狀態

### 階段 6：建立 VM

- [ ] 建立新 VM（Ubuntu Linux 64-bit）
- [ ] 設定規格（16 vCPU / 64 GB RAM / 150 GB Disk）
- [ ] 設定 Boot Firmware 為 **EFI**
- [ ] 加入 PCI Device（Passthrough GPU）
- [ ] 設定 MMIO 參數（**H100/H200 數值需查官方文件確認，官方表格目前未列出**；A100 80GB 為 256 GB，可作為估算參考）：
  - `pciPassthru.64bitMMIOSizeGB = <依 GPU 型號確認>`
  - `pciPassthru.use64bitMMIO = TRUE`

### 階段 7：安裝 Ubuntu 和 GPU Driver

- [ ] 掛載 Ubuntu 22.04 LTS ISO 安裝 Guest OS
- [ ] 安裝 build-essential
- [ ] 下載 NVIDIA Data Center GPU Driver
- [ ] 安裝 Driver 並重開機
- [ ] 執行 `nvidia-smi` 確認 GPU 可見

### 階段 8：安裝 Docker 和 Container Toolkit

- [ ] 安裝 Docker Engine（依 Docker 官方文件）
- [ ] 安裝 NVIDIA Container Toolkit
- [ ] 設定 Docker 使用 NVIDIA Container Runtime
- [ ] 執行驗證容器測試

### 驗證里程碑

- [ ] `nvidia-smi` 在 VM 中顯示 GPU ✅
- [ ] `docker run --gpus all nvidia/cuda:... nvidia-smi` 成功 ✅
- [ ] GPU 使用率可透過 `nvidia-smi dmon` 監控 ✅

### 選用：GPU 監控堆疊（DCGM + Prometheus + Grafana）

- [ ] DCGM 安裝並啟動：`sudo systemctl status nvidia-dcgm` 顯示 active ✅
- [ ] `dcgmi discovery -l` 顯示 GPU ✅
- [ ] DCGM Exporter 容器啟動：`curl http://localhost:9400/metrics` 有輸出 ✅
- [ ] Prometheus + Grafana 透過 Docker Compose 啟動 ✅
- [ ] Grafana 儀表板（Dashboard ID 12239）顯示 GPU 使用率時序圖 ✅

> 詳細步驟參見：[08-NVAIE-GPU-Monitoring.md](08-NVAIE-GPU-Monitoring.md)

---

## 情境二：VMware VM + vGPU（含 License Server）

### 階段 1-2：同情境一（環境準備 + BIOS 設定）

### 階段 3：安裝 VMware ESXi（同情境一）

### 階段 4：安裝 NVAIE Host Software（VIB）

- [ ] 登入 NGC，下載 `NVIDIA-AIE_ESXi_<version>.vib`
- [ ] 上傳 VIB 至 ESXi Datastore
- [ ] 將 ESXi Host 進入 Maintenance Mode
- [ ] 關閉所有 VM
- [ ] 安裝 VIB：`esxcli software vib install -v /vmfs/volumes/.../NVIDIA-AIE_ESXi_<ver>.vib`
- [ ] 重開機 ESXi Host
- [ ] 驗證：`nvidia-smi` 顯示所有 GPU

### 階段 5：設定 Graphics Type

- [ ] 在 vCenter：Host → Configure → Graphics → Host Graphics → Edit
- [ ] 選擇 **Shared Direct**
- [ ] 重開機或重啟 Xorg / nv-hostengine 服務

### 階段 6：部署 DLS License Server

- [ ] 從 NVIDIA Licensing Portal 下載 DLS VM 映像檔
- [ ] 在 VMware 部署 DLS OVA/OVF
- [ ] 設定 DLS VM 靜態 IP（建議獨立 VM，4 vCPU / 8 GB RAM / 15 GB Disk）
- [ ] 存取 DLS 管理介面（https://<DLS-IP>）
- [ ] 完成系統管理員帳號設定
- [ ] 從 NVIDIA Licensing Portal 匯入授權至 DLS
- [ ] 建立 Service Instance
- [ ] **產生 Client Configuration Token**（.tok 檔案）
- [ ] 下載 Token 備用

### 階段 7：建立 vGPU VM

- [ ] 建立新 VM（Ubuntu Linux 64-bit）
- [ ] 設定規格（16 vCPU / 64 GB RAM / 150 GB Disk）
- [ ] 設定 Boot Firmware 為 **EFI**
- [ ] 加入 PCI Device → 選擇 vGPU Profile（C-series；H100 PCIe 例如 `H100-20C`，H100 SXM5 例如 `H100XM-20C`，H200 PCIe 141GB (H200 NVL) 例如 `H200-35C`；完整清單見 04 文件 §4.3）
- [ ] 啟動 VM

### 階段 8：安裝 Ubuntu 和 vGPU Guest Driver

- [ ] 安裝 Ubuntu 22.04 LTS（同 Passthrough 流程）
- [ ] 安裝 build-essential
- [ ] 從 NGC 下載 vGPU Guest Driver（帶 -grid 後綴）
- [ ] 安裝 Grid Driver 並重開機
- [ ] 執行 `nvidia-smi` 確認 vGPU 可見

### 階段 9：套用 vGPU 授權

- [ ] 建立 Token 目錄：`sudo mkdir -p /etc/nvidia/ClientConfigToken/`
- [ ] 複製 .tok 檔案到目錄中
- [ ] 設定正確權限：`sudo chmod 744 /etc/nvidia/ClientConfigToken/*.tok`
- [ ] 重啟授權服務：`sudo systemctl restart nvidia-gridd`
- [ ] 確認授權狀態：`nvidia-smi -q | grep "Licensed"`

### 階段 10：安裝 Docker 和 Container Toolkit（同情境一 階段 8）

### 驗證里程碑

- [ ] `nvidia-smi` 在 VM 中顯示 vGPU ✅
- [ ] 授權狀態顯示 Licensed ✅
- [ ] `docker run --gpus all nvidia/cuda:... nvidia-smi` 成功 ✅
- [ ] DLS 管理介面顯示授權已發放 ✅

### vGPU 監控驗證（nvidia-smi，Guest VM 內）

> ⚠️ DCGM 在 vGPU Guest VM **不受 NVAIE 支援**，請使用 nvidia-smi 進行監控驗證。

- [ ] `nvidia-smi` 顯示 vGPU Framebuffer 使用量與 Compute 使用率 ✅
- [ ] `nvidia-smi dmon -s u` 持續輸出 vGPU 使用率數值 ✅
- [ ] `nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv` 輸出正常 ✅

### vGPU 監控驗證（nvidia-smi，ESXi Host 層）

> Host 層分兩個視角：`nvidia-smi`（無子命令）查看**物理 GPU** 溫度/功耗；`nvidia-smi vgpu` 系列查看**各 vGPU 實例** FB/SM/encoder/decoder 使用率。

- [ ] 在 ESXi Host 執行 `nvidia-smi` 顯示物理 GPU 溫度與功耗 ✅
- [ ] `nvidia-smi vgpu` 顯示所有 vGPU 實例清單 ✅
- [ ] `nvidia-smi vgpu -q` 或 `nvidia-smi vgpu -u` 輸出各實例 FB 使用量與 SM/encoder/decoder 使用率 ✅

> DCGM + Prometheus + Grafana 完整堆疊請改在 **Passthrough VM** 部署，參見：[08-NVAIE-GPU-Monitoring.md](08-NVAIE-GPU-Monitoring.md)

---

## 兩種模式對比快速參考

| 對比項目 | Passthrough | vGPU |
|----------|-------------|------|
| 安裝 VIB | ❌ 不需要 | ✅ 必要 |
| NVAIE 訂閱授權 | ✅ 需要（per-GPU） | ✅ 需要（per-GPU） |
| License Server (NLS) | ❌ 不需要 | ✅ 必要（DLS/CLS） |
| Driver 類型 | Data Center Driver | Grid Guest Driver |
| HGX 多 GPU per VM | 可多 GPU | vSphere 上支援 1/2/4/8 GPU per VM |
| VM 數量/單 GPU | 1 | 最多 20（H100 PCIe 用 H100-4C） |
| Live Migration | ❌ | ✅ |
| 性能 | 最佳（接近裸機） | 略低 |
| 適用場景 | 單一高性能 AI 工作 | 多租戶/多 VM 共享 |

---

## 常用文件連結速查

| 文件 | URL |
|------|-----|
| VMware 部署指南（主頁） | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/index.html |
| 先決條件 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/prereqs.html |
| VIB 安裝 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/vgpu.html |
| License System | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/nls.html |
| 進階 GPU 設定 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/advance-gpu.html |
| 建立第一台 VM | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/first-vm.html |
| Docker 安裝 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/docker.html |
| 驗證 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/validation.html |
| NVIDIA License System | https://docs.nvidia.com/license-system/latest/ |
| Container Toolkit | https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html |
| Support Matrix 8.1 | https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html |
| Support Matrix 7.5 LTSB | https://docs.nvidia.com/ai-enterprise/release-7/7.5/support/support-matrix.html |
| H100/H200 vGPU Types | https://docs.nvidia.com/ai-enterprise/release-7/7.5/infra-software/vgpu/reference/hopper.html |
