# 05 — Ubuntu 24.04 安裝、vGPU Guest Driver、套用 DLS Token、Docker / Container Toolkit / NGC CLI

> 對應流程：階段 5（安裝 Ubuntu 24.04）+ 階段 6（安裝 vGPU Guest Driver、套用 Token、容器工具）
> 完成本章節後，兩個 VM 內 `nvidia-smi` 都應顯示 `NVIDIA H200-70C`、`License Status: Licensed`，且 `docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi` 在容器內可看到同一張 vGPU。
>
> 本章節在兩個 VM 上**重複執行**，操作完全一致。建議先把 `vgpu-test-01` 從頭到尾走完並通過驗證，再對 `vgpu-test-02` 重複。
>
> **本章節指令風格與 [`GPU-passthrough-sop.md`](GPU-passthrough-sop.md) 對齊：採 `sudo -s` 進 root prompt 後執行**，差異點 = 使用 **`-grid` 後綴的 vGPU Guest Driver `.deb`** 並且要**套用 DLS Token**（passthrough 環境不需要 Token）。

---

## 1. 安裝 Ubuntu Server 24.04 LTS

### 1.1 開機進入 Installer

1. vCenter 對 `vgpu-test-01` → **Power On** → **Launch Web Console**。
2. Ubuntu installer 開機畫面 → 選 **Try or Install Ubuntu Server**。

### 1.2 Installer 設定重點

| 步驟 | 設定 |
|------|------|
| Language | English（或依需求） |
| Keyboard | 對應你的鍵盤 |
| Type of installation | `Ubuntu Server`（不要選 `Ubuntu Server (minimized)`） |
| Network | 設定**靜態 IP**（與 DLS 同網段或可達） |
| Proxy | 視環境設定 |
| Mirror | 預設 |
| Storage | `Use an entire disk` → 預設 ext4 / LVM |
| Profile | 建立管理者帳號（例：`ubuntu` / 強密碼） |
| **SSH Setup** | **勾選 `Install OpenSSH server`**（後續才能 SSH 連線） |
| Featured snaps | 不需勾選 |

### 1.3 安裝完成 → 拔除 ISO → 重開機 → SSH 連線

1. Installer 結束選 **Reboot Now**。
2. vCenter VM → **Edit Settings → CD/DVD drive → Disconnect**。
3. 從外部 SSH：`ssh ubuntu@<vm-ip>`。

---

## 2. 作業系統更新

```bash
ubuntu@vgpu-test-01:~$ sudo -s
root@vgpu-test-01:/home/ubuntu# apt update
root@vgpu-test-01:/home/ubuntu# apt upgrade -y

root@vgpu-test-01:/home/ubuntu# uname -a
Linux vgpu-test-01 6.8.0-XX-generic #XX-Ubuntu SMP ... x86_64 x86_64 x86_64 GNU/Linux
root@vgpu-test-01:/home/ubuntu# cat /etc/os-release
root@vgpu-test-01:/home/ubuntu# hostnamectl
```

> 後續操作皆以 `root` prompt 為例，與 passthrough SOP 統一風格。

---

## 3. 停用系統內建 GPU 驅動（nouveau）

```bash
# 1. 確認 vGPU 已被 OS 認到（PCI 列表）
root@vgpu-test-01:/home/ubuntu# lspci | grep -i nvidia
02:01.0 3D controller: NVIDIA Corporation Device <id> (rev a1)

# 2. 確認 nouveau 目前載入中（有輸出代表啟用）
root@vgpu-test-01:/home/ubuntu# lsmod | grep -i nouveau
nouveau              3096576  0
...

# 3. 編輯 /etc/modprobe.d/blacklist.conf，在最下面加入兩行
root@vgpu-test-01:/home/ubuntu# vim /etc/modprobe.d/blacklist.conf
#（在檔尾追加）
blacklist nouveau
options nouveau modeset=0

# 4. 確認加入成功
root@vgpu-test-01:/home/ubuntu# tail -n 5 /etc/modprobe.d/blacklist.conf
# really needed.
blacklist amd76x_edac
blacklist nouveau
options nouveau modeset=0

# 5. 更新 initramfs，並重開機讓 blacklist 生效
root@vgpu-test-01:/home/ubuntu# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.8.0-XX-generic

root@vgpu-test-01:/home/ubuntu# reboot
```

重連 SSH 後驗證：

```bash
ubuntu@vgpu-test-01:~$ sudo -s
root@vgpu-test-01:/home/ubuntu# lsmod | grep -i nouveau
root@vgpu-test-01:/home/ubuntu#       <-- 無輸出，代表 nouveau 已停用
```

---

## 4. 上傳並安裝 vGPU Guest Driver（`.deb` 安裝法）

> ⚠️ **與 passthrough SOP 的核心差異**：
> - passthrough 環境用的是 **Data Center Driver**（例：`nvidia-driver-local-repo-ubuntu2404-595.71.05_1.0-1_amd64.deb`，本機 repo 形式，需後續 `apt install cuda-drivers`）
> - vGPU 環境用的是 **vGPU Guest Driver**（檔名帶 `-grid` 或包含 `grid` 字樣，**直接安裝形式**的 `.deb`，無需另外 `apt install cuda-drivers`）
>
> **絕對不能裝錯**：在 vGPU VM 內裝 Data Center Driver 會抓不到 vGPU，反之亦然。

### 4.1 從 NGC 取得 vGPU Guest Driver ZIP

1. 在管理機登入 NGC，至 **Catalog → vgpu-guest-driver-20**（對應 NVAIE 8.1）下載 `NVIDIA-GRID-Linux-KVM-595.<patch>-vGPU20.<n>.zip`。
2. 解壓 ZIP，於 `Guest_Drivers/` 目錄中取出：
   - `nvidia-linux-grid-595_595.<patch>_amd64.deb`（Ubuntu 24.04 用，本手冊主用）
   - `NVIDIA-Linux-x86_64-595.<patch>-grid.run`（備用）

### 4.2 上傳 `.deb` 至 VM

從管理機（Windows PowerShell）：

```powershell
scp .\nvidia-linux-grid-595_595.<patch>_amd64.deb ubuntu@<vm-ip>:~/
```

在 VM 內檢查：

```bash
root@vgpu-test-01:/home/ubuntu# ls -lh nvidia-linux-grid-595_595.*_amd64.deb
-rw-r--r-- 1 ubuntu ubuntu  ... nvidia-linux-grid-595_595.71.05_amd64.deb
```

### 4.3 安裝 Driver

```bash
# 先確保 apt 索引是新的（停用 nouveau 重開機後第一件事）
root@vgpu-test-01:/home/ubuntu# apt update

# 用 dpkg 安裝（與 passthrough SOP 的 dpkg -i 風格一致）
root@vgpu-test-01:/home/ubuntu# dpkg -i nvidia-linux-grid-595_595.71.05_amd64.deb

# 若 dpkg 報缺依賴，用 apt 補（vGPU deb 通常會帶齊依賴，但保險動作）
root@vgpu-test-01:/home/ubuntu# apt-get install -f -y

# 驗證 deb 安裝結果
root@vgpu-test-01:/home/ubuntu# dpkg -l | grep -i nvidia
ii  nvidia-linux-grid-595         595.71.05-... amd64  NVIDIA vGPU guest driver ...
```

> **不要**像 passthrough 一樣去 `apt install cuda-drivers`；vGPU Guest Driver deb 已含所有必要模組與 `nvidia-gridd` 服務。

### 4.4 重開機並驗證

```bash
root@vgpu-test-01:/home/ubuntu# reboot

# 重連 SSH
ubuntu@vgpu-test-01:~$ sudo -s
root@vgpu-test-01:/home/ubuntu# nvidia-smi
```

預期輸出（樣式示意）：

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.71.05            Driver Version: 595.71.05    CUDA Version: 13.x         |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                                | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap  |          Memory-Usage  | GPU-Util  Compute M. |
|=========================================+========================+======================|
|   0  NVIDIA H200-70C                On   |  00000000:02:01.0 Off  |                    0 |
| N/A  N/A    P0             N/A / N/A     |      0MiB / 72704MiB   |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

**重點檢查：**
- **Name = `NVIDIA H200-70C`**（vGPU profile，不是 `NVIDIA H200 NVL`）
- **Memory total ≈ 72704 MiB**（官方 profile 容量 = **70 GB**；`nvidia-smi` 顯示的 MiB 為實際暴露給 OS 的 Frame Buffer，與 70 GB 略有換算差異屬正常）
- **Power / Temp = `N/A`**（vGPU Guest 看不到實體電壓 / 溫度，正常）

### 4.5 此時授權狀態（預期未授權）

```bash
root@vgpu-test-01:/home/ubuntu# nvidia-smi -q | grep -A 5 "vGPU Software Licensed"
    vGPU Software Licensed Product
        Product Name              : NVIDIA AI Enterprise
        License Status            : Unlicensed (Restricted)
```

> 未授權時 vGPU 仍可運作但會被**降頻 / 功能限制**。完整功能需完成 §5–§6 套用 Token。

---

## 5. 將 DLS Client Configuration Token 複製到 VM

從管理機把章節 03 §7 下載的 `.tok` 傳上 VM：

```powershell
# Windows PowerShell（管理機）
scp .\client_configuration_token_<timestamp>.tok ubuntu@<vm-ip>:~/
```

VM 內：

```bash
root@vgpu-test-01:/home/ubuntu# ls -lh client_configuration_token_*.tok
-rw-r--r-- 1 ubuntu ubuntu  ... client_configuration_token_<timestamp>.tok
```

---

## 6. 套用 Token 並驗證授權

```bash
# 6.1 建立 Token 目錄
root@vgpu-test-01:/home/ubuntu# mkdir -p /etc/nvidia/ClientConfigToken/

# 6.2 複製 Token
root@vgpu-test-01:/home/ubuntu# cp client_configuration_token_*.tok /etc/nvidia/ClientConfigToken/

# 6.3 設定權限（NVIDIA 要求 readable by nvidia-gridd / root）
root@vgpu-test-01:/home/ubuntu# chmod 744 /etc/nvidia/ClientConfigToken/*.tok

# 6.4 確認 nvidia-gridd 服務存在
root@vgpu-test-01:/home/ubuntu# systemctl list-unit-files | grep nvidia-gridd
nvidia-gridd.service   enabled

# 6.5 重啟服務以套用 Token
root@vgpu-test-01:/home/ubuntu# systemctl restart nvidia-gridd
root@vgpu-test-01:/home/ubuntu# systemctl status nvidia-gridd --no-pager

# 6.6 看 journal 確認領到授權
root@vgpu-test-01:/home/ubuntu# journalctl -u nvidia-gridd --no-pager | tail -20
... NVIDIA: License acquired successfully.

# 6.7 nvidia-smi 確認授權狀態
root@vgpu-test-01:/home/ubuntu# nvidia-smi -q | grep -A 5 "vGPU Software Licensed"
    vGPU Software Licensed Product
        Product Name              : NVIDIA AI Enterprise
        License Status            : Licensed (Expiry: YYYY-MM-DD HH:MM:SS)
```

授權必須為 **`Licensed`** 並顯示到期日。同步至 DLS Web UI → **DASHBOARD** 應看到 `1 LEASED`（兩個 VM 都完成後變 `2 LEASED`）。

---

## 7. 安裝 Docker（與 passthrough SOP 同步驟）

> 本章節以下指令直接對齊 [`GPU-passthrough-sop.md`](GPU-passthrough-sop.md) §6 風格，只將主機名替換為 `vgpu-test-01`。

### 7.1 移除可能衝突的舊套件（Ubuntu 24.04 預設通常沒有）

```bash
root@vgpu-test-01:/home/ubuntu# apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```

### 7.2 設定 Docker 的 apt repository（deb822 格式）

```bash
# 新增 gpg key
root@vgpu-test-01:/home/ubuntu# apt update
root@vgpu-test-01:/home/ubuntu# apt install -y ca-certificates curl
root@vgpu-test-01:/home/ubuntu# install -m 0755 -d /etc/apt/keyrings
root@vgpu-test-01:/home/ubuntu# ls -ald /etc/apt/keyrings
drwxr-xr-x 2 root root 6 Mar 31  2024 /etc/apt/keyrings

root@vgpu-test-01:/home/ubuntu# curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
root@vgpu-test-01:/home/ubuntu# chmod a+r /etc/apt/keyrings/docker.asc
root@vgpu-test-01:/home/ubuntu# ls -al /etc/apt/keyrings/docker.asc
-rw-r--r-- 1 root root 3817 ... /etc/apt/keyrings/docker.asc

# 加入 apt repository（deb822 格式，與 passthrough SOP 一致）
root@vgpu-test-01:/home/ubuntu# tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

root@vgpu-test-01:/home/ubuntu# apt update
```

### 7.3 安裝 Docker 與基本驗證

```bash
root@vgpu-test-01:/home/ubuntu# apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
root@vgpu-test-01:/home/ubuntu# systemctl status docker --no-pager
root@vgpu-test-01:/home/ubuntu# docker run hello-world
```

---

## 8. 安裝 NVIDIA Container Toolkit（與 passthrough SOP 同步驟）

### 8.1 前置套件

```bash
root@vgpu-test-01:/home/ubuntu# apt-get update && apt-get install -y --no-install-recommends \
  curl \
  gnupg2
```

### 8.2 設定 Container Toolkit 的 apt repository

```bash
root@vgpu-test-01:/home/ubuntu# curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

root@vgpu-test-01:/home/ubuntu# apt-get update
```

### 8.3 確認可用版本並安裝（固定版本，與 passthrough SOP 同樣模式）

```bash
# 查目前 repo 提供的版本清單
root@vgpu-test-01:/home/ubuntu# apt-cache madison nvidia-container-toolkit
nvidia-container-toolkit | 1.19.1-1 | https://nvidia.github.io/libnvidia-container/stable/deb/amd64  Packages
nvidia-container-toolkit | 1.19.0-1 | https://nvidia.github.io/libnvidia-container/stable/deb/amd64  Packages
nvidia-container-toolkit | 1.18.2-1 | https://nvidia.github.io/libnvidia-container/stable/deb/amd64  Packages
...

# 固定版本安裝
# ‣ 生產／需符合 NVAIE 8.1 Support Matrix → pin 1.19.0-1（本手冊預設）
# ‣ LAB／願意接受 caveat 用上游最新版 → 改成 apt-cache madison 看到的最新版（例 1.19.1-1）
root@vgpu-test-01:/home/ubuntu# export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.19.0-1
root@vgpu-test-01:/home/ubuntu# apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

> **版本選擇 caveat：**
> - [NVAIE 8.1 Support Matrix](https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html) 列示支援的 NVIDIA Container Toolkit 為 **1.19.0**。生產或需要原廠 NVAIE 支援承諾時，**應 pin `1.19.0-1`**。
> - [NVIDIA Container Toolkit Install Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) 與 Release Notes 已示範更新版（如 1.19.1）；功能上多半向後相容，**但若超出 NVAIE Support Matrix 列示版本，原廠不保證在 NVAIE 環境的支援**。LAB 環境可採最新版，但須自行接受此 caveat。

---

## 9. 讓 Docker 使用 NVIDIA Container Runtime

```bash
root@vgpu-test-01:/home/ubuntu# nvidia-ctk runtime configure --runtime=docker
root@vgpu-test-01:/home/ubuntu# systemctl restart docker
```

### 9.1 用容器測試能存取 vGPU

```bash
root@vgpu-test-01:/home/ubuntu# docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

預期輸出：容器內 `nvidia-smi` 顯示與 §4.4 一致的 `NVIDIA H200-70C`（官方 profile 容量 **70 GB**；`nvidia-smi` 可能顯示約 72704 MiB）。若顯示「NVIDIA-SMI has failed」→ 跳 [07-troubleshooting.md](07-troubleshooting.md) §4／§5。

---

## 10. 安裝 NGC CLI（與 passthrough SOP 同步驟，可選但建議）

> 用途：之後拉取 NVAIE 容器（NIM / NeMo / Triton 等）或 vGPU 工件均需要 NGC API Key；NGC CLI 是官方建議的本機客戶端。

### 10.1 上傳 `ngccli_linux.zip` 並建立工作目錄

```bash
# 從管理機 scp 上去（或在 VM 內 wget 下載最新版）
# Windows PS:  scp .\ngccli_linux.zip ubuntu@<vm-ip>:~/

root@vgpu-test-01:/home/ubuntu# mkdir ngccli
root@vgpu-test-01:/home/ubuntu# cd ngccli
root@vgpu-test-01:/home/ubuntu/ngccli# mv ../ngccli_linux.zip ./
root@vgpu-test-01:/home/ubuntu/ngccli# ls
ngccli_linux.zip
```

### 10.2 解壓縮（先裝 unzip）

```bash
root@vgpu-test-01:/home/ubuntu/ngccli# apt-get install -y unzip
root@vgpu-test-01:/home/ubuntu/ngccli# unzip ngccli_linux.zip
```

### 10.3 校驗檔案完整性

```bash
root@vgpu-test-01:/home/ubuntu/ngccli# find ngc-cli/ -type f -exec md5sum {} + | LC_ALL=C sort | md5sum -c ngc-cli.md5
-: OK

root@vgpu-test-01:/home/ubuntu/ngccli# sha256sum ngccli_linux.zip
# 將輸出值與 NGC CLI 下載頁面提供的 SHA256 比對，必須相同
```

### 10.4 設定 PATH

```bash
root@vgpu-test-01:/home/ubuntu/ngccli# chmod u+x ngc-cli/ngc
root@vgpu-test-01:/home/ubuntu/ngccli# echo "export PATH=\"\$PATH:$(pwd)/ngc-cli\"" >> ~/.bash_profile && source ~/.bash_profile
```

### 10.5 驗證

```bash
root@vgpu-test-01:/home/ubuntu/ngccli# ngc --version
NGC CLI 4.x.x
```

### 10.6 串接 NGC API Key

> 先至 https://catalog.ngc.nvidia.com/ → 右上 Account Settings → Keys & Secrets → Generate Personal Key 取得（章節 01 §4 已說明）。

```bash
root@vgpu-test-01:/home/ubuntu/ngccli# ngc config set
Enter API key [no-apikey]. Choices: [<VALID_APIKEY>, 'no-apikey']: <貼上 Personal API Key>
Enter CLI output format type [ascii]. Choices: ['ascii', 'csv', 'json']: json
Enter org [no-org]. Choices: ['<your-org-id>']: <your-org-id>
Enter team [no-team]. Choices: ['no-team']: no-team
Enter ace [no-ace]. Choices: ['no-ace']: no-ace

Validating configuration...
Successfully validated configuration.
Saving configuration...
Successfully saved NGC configuration to /root/.ngc/config
```

### 10.7 讓 Docker CLI 也登入 nvcr.io

```bash
root@vgpu-test-01:/home/ubuntu/ngccli# docker login nvcr.io --username '$oauthtoken'
Password: <貼上 Personal API Key>
...
Login Succeeded
```

---

## 11. 對 VM #2（`vgpu-test-02`）重複本章節

§1 → §10 全部對 `vgpu-test-02` 再執行一次：

- 同一份 `nvidia-linux-grid-595_*.deb`
- 同一份 `client_configuration_token_*.tok`（Token 可給多 VM 共用）
- 同一份 `ngccli_linux.zip`
- 同一組 NGC API Key（同一個帳號可在多台機器登入）

完成後：

| VM | Driver | GPU Name | License Status | Docker GPU test |
|----|--------|----------|----------------|-----------------|
| `vgpu-test-01` | 595.71.05 | NVIDIA H200-70C（官方 profile **70 GB**；`nvidia-smi` 約 72704 MiB） | Licensed | ✅ |
| `vgpu-test-02` | 595.71.05 | NVIDIA H200-70C（官方 profile **70 GB**；`nvidia-smi` 約 72704 MiB） | Licensed | ✅ |

---

## 12. 完成檢查清單（每個 VM 都要逐項對）

- [ ] Ubuntu 24.04 LTS 已安裝，可 SSH 登入
- [ ] `lsmod | grep nouveau` 無輸出（nouveau 已停用）
- [ ] `dpkg -l | grep -i nvidia` 顯示 `nvidia-linux-grid-595`（vGPU Guest Driver，**非** Data Center Driver）
- [ ] `nvidia-smi` 顯示 `NVIDIA H200-70C`，官方 profile 容量 **70 GB**（`nvidia-smi` 可能顯示約 72704 MiB，屬換算差異）
- [ ] `/etc/nvidia/ClientConfigToken/` 有 `.tok` 檔且權限 744
- [ ] `systemctl status nvidia-gridd` = active (running)
- [ ] `nvidia-smi -q | grep "License Status"` = `Licensed`
- [ ] `docker run hello-world` 成功
- [ ] `docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi` 容器內看到 H200-70C
- [ ] `ngc --version` / `ngc config current` 正常
- [ ] `docker login nvcr.io` Login Succeeded

→ 兩個 VM 都 ✅ 後，前往 [06-validation.md](06-validation.md) 進行 Host / DLS / Guest 三方完整最終驗證。
