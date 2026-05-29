# NVAIE GPU 監控：DCGM + Prometheus + Grafana

> 整理日期：2026-05-21  
> 目標環境：H100/H200 GPU、VMware vSphere 8.x、Ubuntu 22.04 VM、Docker  
> 適用 NVAIE 版本：7.5 LTSB / 8.1

---

## 1. NVAIE 包含 GPU 監控嗎？

**是（間接包含）**。NVAIE Infrastructure Layer 的 **Data Center GPU Driver** 安裝後，DCGM 套件即可透過 CUDA apt 套件庫安裝，且 NVIDIA 官方將 DCGM 定為企業級 GPU 監控的標準方案。

| 工具 | 類型 | 用途 |
|------|------|------|
| `nvidia-smi` | 內建（Driver 附帶） | 瞬間快照，手動執行 |
| **DCGM** | 需另行安裝（CUDA 套件庫） | 持續收集、歷史資料、健康診斷 |
| **DCGM Exporter** | Docker 映像（NGC） | 將 DCGM 指標轉為 Prometheus 格式 |
| Prometheus | 開源（第三方） | 時序資料庫 |
| Grafana | 開源（第三方） | 視覺化儀表板 |

> **nvidia-smi vs DCGM**：`nvidia-smi` 是瞬間快照，每次執行才抓一次數據；DCGM 是持續執行的 daemon，可保存歷史、計算統計、偵測健康異常，適合生產環境長期監控。

---

## 2. 建議監控架構

> **⚠️ 適用範圍：Passthrough VM 專用**  
> DCGM + Prometheus + Grafana 完整堆疊僅適用 **GPU Passthrough** 虛擬機。  
> vGPU Guest VM **不支援 DCGM**（NVAIE 8.1 / 7.5 官方限制），vGPU 監控方式見第 4 章。

```
GPU Hardware（H100/H200 — Passthrough VM 專用）
         │
         ▼
  DCGM daemon（dcgm-hostengine）      ← 安裝於 Passthrough Ubuntu VM
         │
         ▼
  DCGM Exporter（Docker，port 9400）  ← 轉換成 Prometheus 格式
         │
         ▼
  Prometheus（Docker，port 9090）      ← 時序資料庫
         │
         ▼
  Grafana（Docker，port 3000）         ← 視覺化儀表板
```

所有元件均執行於 **Passthrough Ubuntu VM 內**，與 VMware vSphere / ESXi Host 無關。

---

## 3. 可監控的 GPU 指標

> 以下指標適用 **Passthrough VM 上的 DCGM**（Data Center GPU Driver 完整支援）。  
> vGPU Guest VM 的可見指標有嚴格限制，詳見第 4 章。

### 3.1 效能指標

| 指標 | DCGM 欄位 | 說明 |
|------|-----------|------|
| GPU 使用率 | `DCGM_FI_DEV_GPU_UTIL` | GPU 整體忙碌百分比（0-100%）|
| SM 使用率 | `DCGM_FI_PROF_SM_ACTIVE` | Streaming Multiprocessor 活躍比例 |
| 記憶體使用量 | `DCGM_FI_DEV_FB_USED` | 已用 GPU 記憶體（MB）|
| 記憶體總量 | `DCGM_FI_DEV_FB_TOTAL` | GPU 總記憶體（MB）|
| 記憶體頻寬 | `DCGM_FI_PROF_DRAM_ACTIVE` | DRAM 讀寫活躍比例 |
| SM 時脈 | `DCGM_FI_DEV_SM_CLOCK` | Streaming Multiprocessor 時脈（MHz）|
| 記憶體時脈 | `DCGM_FI_DEV_MEM_CLOCK` | 記憶體時脈（MHz）|

### 3.2 電力與溫度

| 指標 | DCGM 欄位 | 說明 |
|------|-----------|------|
| GPU 溫度 | `DCGM_FI_DEV_GPU_TEMP` | GPU 核心溫度（°C）|
| 記憶體溫度 | `DCGM_FI_DEV_MEMORY_TEMP` | HBM 記憶體溫度（°C）|
| 功耗 | `DCGM_FI_DEV_POWER_USAGE` | 即時功耗（W）|
| 功耗上限 | `DCGM_FI_DEV_POWER_MGMT_LIMIT` | 設定的功耗上限（W）|

### 3.3 健康與錯誤指標

| 指標 | DCGM 欄位 | 說明 |
|------|-----------|------|
| 單位元 ECC 錯誤 | `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` | 單位元錯誤（可自動修正） |
| 雙位元 ECC 錯誤 | `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL` | 雙位元錯誤（嚴重，需調查）|
| XID 錯誤 | `DCGM_FI_DEV_XID_ERRORS` | 驅動程式/硬體錯誤代碼 |
| PCIe Replay 計數 | `DCGM_FI_DEV_PCIE_REPLAY_COUNTER` | PCIe 傳輸重試次數 |

### 3.4 互連指標（H100/H200 HGX）

| 指標 | DCGM 欄位 | 說明 |
|------|-----------|------|
| NVLink 傳輸速率 | `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL` | NVLink 總頻寬 |

### 3.5 Profiling 指標（進階）

需 DCGM Profiling 模組（DataCenter GPU 支援），提供 Tensor Core、FP32/FP16/FP64 引擎使用率等硬體效能計數器。

---

## 4. vGPU 環境監控限制

### 4.1 DCGM 在 vGPU Guest VM **不受支援**

根據 **NVAIE 8.1 / 7.5 官方文件（Limitations）**：  
DCGM 明確列為 **不支援** 在 vGPU 環境執行，包含：
- vGPU Guest VM（無論 MIG-backed 或 Time-Sliced profile）
- Hypervisor Host（ESXi）

在 vGPU Guest VM 嘗試執行 DCGM 可能出現：指標為 0 或 N/A、Profiling 錯誤、指標遺失等問題，**不可作為監控依據**。

> **結論：** DCGM + Prometheus + Grafana 完整堆疊**僅適用 Passthrough VM**。

---

### 4.2 vGPU Guest VM 的實際可見指標

vGPU Guest VM 透過 **nvidia-smi** 只能看到有限的指標範圍（依 NVIDIA vGPU 文件）：

| 指標 | vGPU Guest VM 可見性 |
|------|----------------------|
| 3D / Compute 使用率 | ✅ 可見 |
| Framebuffer 使用量 | ✅ 可見（受限於 vGPU Profile 分配量）|
| Encoder / Decoder 使用率 | ✅ 可見 |
| Memory Controller 使用率 | ✅ 可見 |
| GPU 溫度 | ❌ 回報 0 或 N/A |
| 功耗（Power Usage） | ❌ 回報 0 或 N/A |
| ECC 錯誤計數 | ❌ 回報 0 或 N/A |
| NVLink 指標 | ❌ 不可見 |
| XID 錯誤 | ❌ 僅 Hypervisor 層可見 |
| PCIe 頻寬 | ❌ 不可見 |

---

### 4.3 vGPU 環境的建議監控方式

**Guest VM 內（nvidia-smi）：**

```bash
# 查看 vGPU 計算/記憶體使用率（guest 可見範圍）
nvidia-smi

# 持續監控 vGPU 使用率
nvidia-smi dmon -s u

# 查看 vGPU Framebuffer 使用量
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu,utilization.memory \
           --format=csv,noheader
```

**ESXi Hypervisor Host 層：**

```bash
# ── 物理 GPU 層（溫度、功耗等硬體指標）──
# 查看所有物理 GPU 的溫度、功耗、時脈等
nvidia-smi

# 持續輸出物理 GPU 硬體指標
nvidia-smi dmon

# ── vGPU 實例層（各 VM 的 FB / SM / Encoder / Decoder 使用率）──
# 列出所有 vGPU 實例
nvidia-smi vgpu

# 查看各 vGPU 實例的 FB 使用量與 SM/encoder/decoder 使用率
nvidia-smi vgpu -q

# 以 engine usage 格式顯示各 vGPU 實例資源使用
nvidia-smi vgpu -u

# 持續監控所有 vGPU 實例
nvidia-smi vgpu -l 5
```

> **重要**：物理 GPU 的溫度與功耗需由 `nvidia-smi`（無 `vgpu` 子命令）查看；  
> `nvidia-smi vgpu -q` / `-u` 則顯示每個 **vGPU 實例**的 FB 使用量、SM 使用率、encoder/decoder 使用率，兩者監控對象不同。

---

## 5. DCGM 安裝（Passthrough Ubuntu VM）

> **前提：**
> - 環境為 **GPU Passthrough VM**（非 vGPU Guest VM）
> - 已安裝 NVIDIA Data Center GPU Driver，且 `nvidia-smi` 正常顯示 GPU
> - 系統已設定 **NVIDIA/CUDA apt 套件庫**（未設定者見下方 5.0）

### 5.0 設定 CUDA apt 套件庫（若尚未設定）

```bash
# 安裝必要工具
sudo apt-get install -y wget gnupg

# 下載並安裝 CUDA keyring（以 Ubuntu 22.04 為例）
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
```

> 其他 Ubuntu 版本請至 https://developer.download.nvidia.com/compute/cuda/repos/ 選擇對應路徑。

### 5.1 安裝 DCGM 套件

```bash
# 取得已安裝的 CUDA 版本號（主版本）
CUDA_VERSION=$(nvidia-smi -q | sed -E -n 's/CUDA Version[ :]+([0-9]+)[.].*/\1/p')
echo "CUDA Version: ${CUDA_VERSION}"

# 確保 apt 套件庫已更新
sudo apt-get update

# 移除舊版（如有）
sudo dpkg --list datacenter-gpu-manager &> /dev/null && \
  sudo apt purge --yes datacenter-gpu-manager

# 安裝 DCGM（對應 CUDA 版本）
sudo apt-get install --yes \
                     --install-recommends \
                     datacenter-gpu-manager-4-cuda${CUDA_VERSION}

# 啟用並啟動 DCGM 服務
sudo systemctl --now enable nvidia-dcgm

# 確認服務狀態
sudo systemctl status nvidia-dcgm
```

### 5.2 基礎 dcgmi 指令驗證

```bash
# 列出所有偵測到的 GPU
dcgmi discovery -l

# 檢查 GPU 健康狀態
dcgmi health -g 0

# 以 JSON 格式顯示健康狀態（適合腳本解析）
dcgmi health -g 0 -j

# 啟用統計資料收集
dcgmi stats -e

# 查看 GPU 統計摘要
dcgmi stats -g 0 -v

# 列出可用的指標欄位（Field ID）
dcgmi dmon --list
```

---

## 6. DCGM Exporter（Docker 容器）

DCGM Exporter 將 DCGM 的指標轉換成 Prometheus 可抓取的 HTTP endpoint（port 9400）。

### 6.1 確認 NGC API Key

DCGM Exporter 映像存放於 `nvcr.io`，需要 NGC API Key（與 NVAIE Driver 下載使用同一組帳號）：

```bash
# 登入 NGC Container Registry
docker login nvcr.io
# Username: $oauthtoken
# Password: <NGC API Key>
```

### 6.2 啟動 DCGM Exporter 容器

```bash
docker run -d \
  --name dcgm-exporter \
  --gpus all \
  --cap-add SYS_ADMIN \
  --restart unless-stopped \
  -p 9400:9400 \
  nvcr.io/nvidia/k8s/dcgm-exporter:4.5.3-4.8.2-distroless
```

### 6.3 驗證 Metrics Endpoint

```bash
# 確認容器正在運行
docker ps | grep dcgm-exporter

# 抓取 Prometheus 格式指標
curl http://localhost:9400/metrics | head -40

# 確認特定指標存在
curl -s http://localhost:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL
```

輸出範例：
```
# HELP DCGM_FI_DEV_GPU_UTIL GPU utilization (in %).
# TYPE DCGM_FI_DEV_GPU_UTIL gauge
DCGM_FI_DEV_GPU_UTIL{gpu="0",UUID="GPU-...",modelName="NVIDIA H100 PCIe"} 42

# HELP DCGM_FI_DEV_FB_USED Framebuffer memory used (in MiB).
# TYPE DCGM_FI_DEV_FB_USED gauge
DCGM_FI_DEV_FB_USED{gpu="0",UUID="GPU-...",modelName="NVIDIA H100 PCIe"} 8192
```

---

## 7. Prometheus + Grafana 部署（Docker Compose）

### 7.1 建立目錄結構

```bash
mkdir -p ~/gpu-monitoring/prometheus
cd ~/gpu-monitoring
```

### 7.2 建立 prometheus.yml

```bash
cat > ~/gpu-monitoring/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'dcgm-exporter'
    static_configs:
      - targets: ['dcgm-exporter:9400']
    scrape_interval: 15s
EOF
```

### 7.3 建立 docker-compose.yml

```bash
cat > ~/gpu-monitoring/docker-compose.yml << 'EOF'
version: '3.8'

services:
  dcgm-exporter:
    image: nvcr.io/nvidia/k8s/dcgm-exporter:4.5.3-4.8.2-distroless
    container_name: dcgm-exporter
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    cap_add:
      - SYS_ADMIN
    restart: unless-stopped
    ports:
      - "9400:9400"

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    depends_on:
      - dcgm-exporter

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
EOF
```

### 7.4 啟動監控堆疊

```bash
cd ~/gpu-monitoring
docker compose up -d

# 確認所有服務啟動
docker compose ps
```

### 7.5 匯入 NVIDIA 官方 Grafana 儀表板

1. 開啟瀏覽器：`http://<VM-IP>:3000`
2. 登入（預設：admin / admin）
3. 左側選單 → **Connections** → **Data Sources** → Add data source → **Prometheus**
   - URL：`http://prometheus:9090`
   - Save & Test
4. 左側選單 → **Dashboards** → **Import**
   - Dashboard ID：**`12239`**（NVIDIA DCGM Exporter Dashboard）
   - 選擇 Prometheus 資料來源 → Import

---

## 8. 常用指令快速參考

### 8.1 nvidia-smi 指令（基礎監控）

```bash
# 即時 GPU 狀態
nvidia-smi

# 持續監控（每 1 秒更新）
nvidia-smi -l 1

# 動態監控模式（顯示多個指標）
nvidia-smi dmon

# 完整詳細資訊
nvidia-smi -q

# 查詢特定指標（CSV 輸出）
nvidia-smi --query-gpu=name,utilization.gpu,utilization.memory,memory.used,memory.total,temperature.gpu,power.draw --format=csv,noheader

# 查看 GPU 拓撲（NVLink 連線）
nvidia-smi topo -m
```

### 8.2 dcgmi 指令（進階監控）

```bash
# 探索：列出所有 GPU
dcgmi discovery -l

# 健康檢查（快速）
dcgmi health -g 0

# 健康診斷（Quick level，約幾秒）
dcgmi diag -r 1

# 健康診斷（Medium level，約 2 分鐘）
dcgmi diag -r 2

# 持續監控特定指標（每 1 秒）
dcgmi dmon -d 1000 -e 150,155,203,204

# 常用欄位 ID 對照
# 150 = GPU Utilization
# 155 = Memory Utilization
# 203 = Temperature
# 204 = Power Usage
# 100 = SM Clock
# 156 = Memory Used (FB)
```

### 8.3 驗證監控堆疊

```bash
# 確認 DCGM 服務
sudo systemctl status nvidia-dcgm

# 確認 DCGM Exporter 正常輸出
curl -s http://localhost:9400/metrics | grep -E "GPU_UTIL|FB_USED|GPU_TEMP|POWER_USAGE"

# 確認 Prometheus 有資料
curl -s 'http://localhost:9090/api/v1/query?query=DCGM_FI_DEV_GPU_UTIL' | python3 -m json.tool
```

---

## 9. 參考連結

| 文件 | URL |
|------|-----|
| DCGM 官方文件 | https://docs.nvidia.com/datacenter/dcgm/latest/ |
| DCGM 安裝指南 | https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/getting-started.html |
| DCGM Field Reference | https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html |
| DCGM Exporter（GitHub） | https://github.com/NVIDIA/dcgm-exporter |
| DCGM Exporter（NGC） | https://catalog.ngc.nvidia.com/orgs/nvidia/teams/k8s/containers/dcgm-exporter |
| Grafana Dashboard 12239 | https://grafana.com/grafana/dashboards/12239 |
| Prometheus 官方文件 | https://prometheus.io/docs/ |
| NGC 登入 | https://ngc.nvidia.com |
