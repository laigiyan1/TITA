# NVAIE Docker 與 NVIDIA Container Toolkit 安裝

> 來源：
> - https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/docker.html  
> - https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html  
> 整理日期：2026-05-20  
> NVIDIA Container Toolkit 最新穩定版（2026-05）：**1.19.0-1**（請安裝時依 apt 取得的最新版本為準）

---

## 1. 概覽

在 Ubuntu VM 中安裝以下組件，讓 Docker 容器可以使用 GPU：

```
[應用容器]
    ↓
[Docker Engine]
    ↓
[NVIDIA Container Toolkit]
    ↓
[NVIDIA Driver (Data Center 或 Grid)]
    ↓
[GPU（Passthrough 或 vGPU）]
```

---

## 2. 安裝 Docker Engine

參考官方 Docker 安裝文件（Ubuntu）：  
https://docs.docker.com/engine/install/ubuntu/

```bash
# 移除舊版 Docker（若有）
sudo apt-get remove docker docker-engine docker.io containerd runc

# 設定 Docker 倉庫
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安裝 Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 確認安裝
sudo docker run hello-world

# 讓目前使用者可執行 Docker（不需 sudo）
sudo usermod -aG docker $USER
newgrp docker
```

---

## 3. 安裝 NVIDIA Container Toolkit

參考官方文件：  
https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

```bash
# 設定 NVIDIA Container Toolkit 倉庫
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 安裝 Container Toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 設定 Docker 使用 NVIDIA Container Runtime
sudo nvidia-ctk runtime configure --runtime=docker

# 重啟 Docker
sudo systemctl restart docker
```

---

## 4. 驗證 GPU 容器可用

```bash
# 測試所有 GPU 可用
sudo docker run --rm --runtime=nvidia --gpus all \
  nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi

# 指定使用 2 個 GPU
docker run --runtime=nvidia --gpus 2 \
  nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi

# 指定特定 GPU（依 device ID）
docker run --runtime=nvidia --gpus '"device=1,2"' \
  nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi
```

---

## 5. 疑難排解

### 5.1 nvidia-container-cli: initialization error

```bash
# 步驟 1：確認 Driver 已安裝
nvidia-smi

# 步驟 2：確認 Container Toolkit 已安裝
which nvidia-container-cli

# 步驟 3：重啟 Docker
sudo systemctl restart docker

# 步驟 4：再次測試
sudo docker run --rm --gpus all nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi
```

### 5.2 CUDA 版本相容性注意事項
- **容器內的 CUDA 版本不能高於 Host Driver 的 CUDA 版本**
- 例如：Host Driver 支援 CUDA 12.8，容器最高可用 CUDA 12.8

---

## 6. 存取 NVAIE 容器映像

NVAIE 容器在 NGC 的私有倉庫，需要 API Key：

```bash
# 產生 NGC API Key（登入 ngc.nvidia.com → 右上角帳號 → Setup → Generate API Key）

# 登入 NVIDIA Container Registry
docker login nvcr.io
# Username: $oauthtoken
# Password: <NGC API Key>

# 或使用環境變數
echo "<NGC API Key>" | docker login nvcr.io --username '$oauthtoken' --password-stdin
```

### 6.1 NVAIE 容器範例

> **版本說明**：以下 image tag（如 `1.3.3`、`25.01`）為整理當時版本，**可能已有更新版本**。部署前請至 [NGC Catalog](https://ngc.nvidia.com/catalog) 查詢對應 image 的最新 tag。

```bash
# NIM 推論微服務（範例，tag 請至 NGC 確認最新版）
docker pull nvcr.io/nim/meta/llama-3.1-8b-instruct:1.3.3

# NeMo 框架（tag 請至 NGC 確認最新版）
docker pull nvcr.io/nvidia/nemo:25.01

# Triton 推論伺服器（tag 請至 NGC 確認最新版）
docker pull nvcr.io/nvidia/tritonserver:25.01-py3

# CUDA 開發環境（tag 請至 NGC 確認最新版）
docker pull nvcr.io/nvidia/cuda:12.8.1-devel-ubuntu22.04
```

---

## 7. NVIDIA Container Toolkit 設定檔

設定檔位置：`/etc/nvidia-container-runtime/config.toml`

關鍵設定項目：
```toml
[nvidia-container-cli]
  # 若使用 vGPU，需確認以下設定
  no-cgroups = false

[nvidia-container-runtime]
  debug = "/var/log/nvidia-container-runtime.log"
```

---

## 8. 參考連結

| 文件 | URL |
|------|-----|
| NVAIE Docker 安裝說明 | https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/docker.html |
| Container Toolkit 安裝 | https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html |
| Sample Workload 驗證 | https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/sample-workload.html |
| Docker 官方 Ubuntu 安裝 | https://docs.docker.com/engine/install/ubuntu/ |
