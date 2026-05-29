---
---
|                                                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **說明**                                                                                                                                                                               |
| 練習在GPU Server中安裝系統，並安裝Docker，可以讓Container可以使用GPU資源。<br>LAB環境中，為了方便練習，在底層裝ESXi，透過Direct Path I/O(Passthrough)方式直接將L4 GPU指派給單一VM，於VM中練習安裝流程。<br>2026/05 - 不使用VM template方式重做一次，以確認完整流程 |

# 參考資料

* 停用作業系統內建的GPU驅動：[Disable Nouveau](https://docs.nvidia.com/ai-enterprise/deployment/bare-metal/latest/nouveau.html#ubuntu)
* 原廠安裝手冊：[NVIDIA AI Enterprise: Bare Metal Deployment Guide](https://docs.nvidia.com/ai-enterprise/deployment/bare-metal/latest/index.html)
* 在Ubuntu中安裝Docker：<https://docs.docker.com/engine/install/ubuntu/>
* 安裝Container Toolkit：<https://docs.nvidia.com/ai-enterprise/deployment/bare-metal/latest/docker.html#installing-the-nvidia-container-toolkit>
* 在Docker中建立一點工作負載，測試能正常使用GPU：<https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/sample-workload.html#running-a-sample-workload>
* NGC CLI的載點及安裝描述：<https://org.ngc.nvidia.com/setup/installers/cli>
	

# 1\. 下載安裝檔

## 1.1 NVIDIA驅動程式

網址：<https://www.nvidia.com/zh-tw/drivers/>
依據GPU型號與要安裝的作業系統版本選擇，網站會推薦最適合的驅動程式

### 1.1.1 本篇LAB環境為Ubuntu 24.04、GPU卡為L4，得到推薦資訊如下：

```
#2026/05/21資訊
Data Center Driver for Ubuntu 24.04 595.71.05 | Linux 64-bit Ubuntu 24.04
驅動程式首頁 > L4 | Linux 64-bit Ubuntu 24.04 > Data Center Driver for Ubuntu 24.04

驅動程式版本: 595.71.05
發佈日期: Tue Apr 28, 2026
作業系統: Linux 64-bit Ubuntu 24.04
CUDA Toolkit: 13.2
語言: English (US)
檔案大小: 783.71 MB
-->下載後檔名為：nvidia-driver-local-repo-ubuntu2404-595.71.05_1.0-1_amd64.deb

#2025/12/22資訊
Data Center Driver for ARM64 Ubuntu 24.04 580.105.08 | Linux ARM 64-bit Ubuntu 24.04  -> 這個錯了，不是ARM64，而應該是AMD64
驅動程式版本: 580.105.08
發佈日期: Thu Nov 06, 2025
作業系統: Linux ARM 64-bit Ubuntu 24.04
CUDA Toolkit: 13.0
語言: English (US)
檔案大小: 601.25 MB
```

### 1.1.2 同場加映，若為H200 NVL，作業系統為Ubuntu 24.04，得到推薦資訊如下(展開查看)

```
Data Center Driver for Ubuntu 24.04 595.71.05 | Linux 64-bit Ubuntu 24.04
驅動程式首頁 > NVIDIA H200 NVL | Linux 64-bit Ubuntu 24.04 > Data Center Driver for Ubuntu 24.04

驅動程式版本: 595.71.05
發佈日期: Tue Apr 28, 2026
作業系統: Linux 64-bit Ubuntu 24.04
CUDA Toolkit: 13.2
語言: English (US)
檔案大小: 783.71 MB
```

## 1.2 NGC(NVIDIA GPU Cloud) CLI

### 1.2.1 連線到NGC Catalog官網

網址為：<https://catalog.ngc.nvidia.com/>

### 1.2.2 於右上角Setup中下載NGC CLI安裝檔

點右上角"Welcome Guest" -> 點擊"Setup" -> 會進入Setup頁面中，於頁面中的Developer Tools中，點擊CLI的"Downloads"進行檔案下載
等待一下子，畫面中會顯示各種作業系統對應到不同NGC CLI版本的安裝檔 -> 這邊選擇AMD64 Linux中的4.10.0版本 (12/10/2025釋出) -> 2026/04/29釋出版本為4.18.0
檔名為ngccli\_linux.zip

# 2\. VMware ESXi中建立VM，並安裝作業系統

## 2.1 VMware ESXi設定啟用GPU Passthrough

1. 登入VMware vCenter，於Inventory介面中點擊ESXi Server
2. 點擊"設定" -> 點擊\[硬體\]類別中的"PCI裝置"
3. 勾選GPU卡 -> 啟用"傳遞(Passthrough)"
4. 第一次啟用Passthrough會需要重啟ESXi生效，重開完就可以自由切換(toggle passthrough)
5. 重開完成後，於相同位置確認GPU卡的\[Passthrough\]欄位顯示為Enabled

## 2.2 建立GPU GuestVM

1. 建立新 VM（Ubuntu Linux 64-bit）
2. 依需求設定規格（16 vCPU / 64 GB RAM / 150 GB Disk）
3. 確認 Boot Firmware 為 **EFI  (此為預設值)**
4. 加入 PCI Device（Passthrough GPU） -> 選擇0000:b5:00:0 DirectPath I/O的裝置
5. 設定 MMIO 參數（**單張L4設定為64，單張H200 NVL設定為128**） 
	* `pciPassthru.64bitMMIOSizeGB = <依 GPU 型號確認>`
	* `pciPassthru.use64bitMMIO = TRUE`
	* \-> 經測試，L4的pciPassthru.64bitMMIOSizeGB設定32會開不起來
		嘗試開機時錯誤訊息：Error message from r760-l4.linkone.lab: The firmware could not allocate 33587200 KB of PCI MMIO. Increase the size of PCI MMIO and try again.
		

## 2.3 Ubuntu作業系統安裝

安裝Ubuntu 24.04，過程不贅述，重點如下

* 選擇安裝Ubuntu Server
* 設定網路，依需求設定proxy
* 設定完會測試連外
* 記得安裝Openssh server，後續才能透過SSH連線

# 3\. 作業系統更新及環境設定

## 3.1 作業系統更新

```
ubuntu@gpuserver:~$ sudo -s
root@gpuserver:/home/ubuntu# apt update
root@gpuserver:/home/ubuntu# apt upgrade

root@gpuserver:/home/ubuntu# uname -a
Linux gpuserver 6.8.0-90-generic #91-Ubuntu SMP PREEMPT_DYNAMIC Tue Nov 18 14:14:30 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
root@gpuserver:/home/ubuntu# cat /etc/os-release
root@gpuserver:/home/ubuntu# hostnamectl


```

## 3.2 停用系統內建GPU驅動

```
ubuntu@gpuserver:~$ sudo -s
root@gpuserver:/home/ubuntu# lspci | grep -i nvidia
03:00.0 3D controller: NVIDIA Corporation AD104GL [L4] (rev a1)
root@gpuserver:/home/ubuntu# lsmod | grep -i nouveau                  ->有顯示資訊就是有啟用
nouveau              3096576  0
mxm_wmi                12288  1 nouveau
drm_gpuvm              45056  1 nouveau
drm_exec              12288  2 drm_gpuvm,nouveau
gpu_sched              61440  1 nouveau
drm_display_helper    237568  1 nouveau
i2c_algo_bit          16384  1 nouveau
video                  77824  1 nouveau
wmi                    28672  3 video,mxm_wmi,nouveau
drm_ttm_helper        12288  2 vmwgfx,nouveau
ttm                  110592  3 vmwgfx,drm_ttm_helper,nouveau

root@gpuserver:/home/ubuntu# vim /etc/modprobe.d/blacklist.conf
#在最下面加入這兩行
blacklist nouveau
options nouveau modeset=0

root@gpuserver:/home/ubuntu# tail -n 5 /etc/modprobe.d/blacklist.conf
# really needed.
blacklist amd76x_edac
blacklist nouveau
options nouveau modeset=0

root@gpuserver:/home/ubuntu# update-initramfs -u
#會顯示以下資訊
update-initramfs: Generating /boot/initrd.img-6.8.0-90-generic

root@gpuserver:/home/ubuntu# reboot

ubuntu@gpuserver:~$ sudo -s
[sudo] password for ubuntu:
root@gpuserver:/home/ubuntu# lsmod | grep -i nouveau
root@gpuserver:/home/ubuntu#
```

# 4\. 安裝NVIDIA驅動程式

## 4.1 將NVIDIA驅動程式上傳進主機中

LAB中的安裝檔檔名為nvidia-driver-local-repo-ubuntu2404-580.105.08\_1.0-1\_amd64.deb
傳入的檔案會在/home/<username>目錄中

## 4.2 用root權限安裝NVIDIA驅動程式

```
ubuntu@gpuserver:~$ sudo -s
root@gpuserver:/home/ubuntu# dpkg -i nvidia-driver-local-repo-ubuntu2404-580.105.08_1.0-1_amd64.deb
root@gpuserver:/home/ubuntu# cp /var/nvidia-driver-local-repo-ubuntu2404-580.105.08/nvidia-driver-local-207F658F-keyring.gpg /usr/share/keyrings/
root@gpuserver:/home/ubuntu# dpkg -l 'nvidia*'
```

# 5\. 安裝CUDA Driver(不是CUDA Toolkit，是底層的CUDA Driver)

CUDA整體架構由下到上分別為CUDA Driver、CUDA Runtime、CUDA Library，CUDA Toolkit是包含了CUDA Runtime及CUDA Library
這邊只是要安裝CUDA Driver而已
```
root@gpuserver:/home/ubuntu# apt install cuda-drivers  -> 沒有apt update會找不到
E: Unable to locate package cuda-drivers
root@gpuserver:/home/ubuntu# apt update
root@gpuserver:/home/ubuntu# apt install cuda-drivers
root@gpuserver:/home/ubuntu# dpkg -l 'cuda*'
root@gpuserver:/home/ubuntu# reboot

root@gpuserver:/home/ubuntu# nvidia-smi
Mon Dec 22 07:18:40 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08            Driver Version: 580.105.08    CUDA Version: 13.0    |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf          Pwr:Usage/Cap |          Memory-Usage | GPU-Util  Compute M. |
|                                        |                        |              MIG M. |
|=========================================+========================+======================|
|  0  NVIDIA L4                      On  |  00000000:03:00.0 Off |                    0 |
| N/A  42C    P8            12W /  72W |      0MiB /  23034MiB |      0%      Default |
|                                        |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU  GI  CI              PID  Type  Process name                        GPU Memory |
|        ID  ID                                                              Usage      |
|=========================================================================================|
|  No running processes found                                                            |
+-----------------------------------------------------------------------------------------+
```

# 6\. 安裝Docker

## 6.1 移除會衝突的套件

Ubuntu 24.04執行時是沒有移除任何套件
```
root@gpuserver:/home/ubuntu# sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```

## 6.2 安裝Docker

### 6.2.1 設定Docker的apt repository

```
#新增gpgkey
root@gpuserver:/home/ubuntu# apt update
root@gpuserver:/home/ubuntu# apt install ca-certificates curl        ->已經裝了
root@gpuserver:/home/ubuntu# install -m 0755 -d /etc/apt/keyrings    ->原本就是755
root@gpuserver:/home/ubuntu# ls -ald /etc/apt/keyrings
drwxr-xr-x 2 root root 6 Mar 31  2024 /etc/apt/keyrings
root@gpuserver:/home/ubuntu# curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
root@gpuserver:/home/ubuntu# chmod a+r /etc/apt/keyrings/docker.asc
root@gpuserver:/home/ubuntu# ls -al /etc/apt/keyrings/docker.asc
-rw-r--r-- 1 root root 3817 Dec 22 07:36 /etc/apt/keyrings/docker.asc

#加入apt repository
root@gpuserver:/home/ubuntu# tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

root@gpuserver:/home/ubuntu# apt update
```

### 6.2.2 安裝Docker，安裝完成後，使用hello-world container驗證

```
root@gpuserver:/home/ubuntu# apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
root@gpuserver:/home/ubuntu# systemctl status docker
root@gpuserver:/home/ubuntu# docker run hello-world
```

# 7\. 安裝Container Toolkit

目前LAB中，可以安裝的NVIDIA Container Toolkit最新版本為1.18.1
若有更新版本，以新版文件為主

## 7.1 安裝前置需求套件

```
root@gpuserver:/home/ubuntu# apt-get update && sudo apt-get install -y --no-install-recommends \
  curl \
  gnupg2
```

## 7.2 設定Container Toolkit的apt repository

```
root@gpuserver:/home/ubuntu# curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

root@gpuserver:/home/ubuntu# apt-get update
```

## 7.3  安裝NVIDIA Container Toolkit

### 7.3.1 先確認NVIDIA Container Toolkit最新版本

<https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/release-notes.html>
目前LAB中，可以安裝的NVIDIA Container Toolkit最新版本為1.18.1
若有更新版本，以新版文件為主
\-> 2026/05/22最新版本為1.19.1-1
```
#這邊的資訊為2026/05/22更新
root@gputestvm:/home/gpusr01# apt-cache madison nvidia-container-toolkit
nvidia-container-toolkit |  1.19.1-1 | https://nvidia.github.io/libnvidia-container/stable/deb/amd64  Packages
nvidia-container-toolkit |  1.19.0-1 | https://nvidia.github.io/libnvidia-container/stable/deb/amd64  Packages
nvidia-container-toolkit |  1.18.2-1 | https://nvidia.github.io/libnvidia-container/stable/deb/amd64  Packages
nvidia-container-toolkit |  1.18.1-1 | https://nvidia.github.io/libnvidia-container/stable/deb/amd64  Packages
...
```

### 7.3.2 根據最新版本資訊，調整指令內容並安裝Container Toolkit

```
root@gpuserver:/home/ubuntu# export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.18.1-1
root@gpuserver:/home/ubuntu# sudo apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

# 8\. 於Docker中調整設定，以存取GPU資源

## 8.1 讓Docker可以使用NVIDIA Container Runtime

```
root@gpuserver:/home/ubuntu# nvidia-ctk runtime configure --runtime=docker
root@gpuserver:/home/ubuntu# systemctl restart docker
```

# 9\. 於Docker中建立工作負載，以測試能正常存取GPU

```
root@gpuserver:/home/ubuntu# docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
#成功資訊如下：
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
20043066d3d5: Pull complete
06808451f0d6: Download complete
Digest: sha256:c35e29c9450151419d9448b0fd75374fec4fff364a27f176fb458d472dfc9e54
Status: Downloaded newer image for ubuntu:latest
Mon Dec 22 08:27:02 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08            Driver Version: 580.105.08    CUDA Version: 13.0    |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf          Pwr:Usage/Cap |          Memory-Usage | GPU-Util  Compute M. |
|                                        |                        |              MIG M. |
|=========================================+========================+======================|
|  0  NVIDIA L4                      On  |  00000000:03:00.0 Off |                    0 |
| N/A  41C    P8            12W /  72W |      0MiB /  23034MiB |      0%      Default |
|                                        |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU  GI  CI              PID  Type  Process name                        GPU Memory |
|        ID  ID                                                              Usage      |
|=========================================================================================|
|  No running processes found                                                            |
+-----------------------------------------------------------------------------------------+
root@gpuserver:/home/ubuntu#
```

# 10\. 安裝NGC CLI

**註：**步驟10.1&10.2可以合併為單一指令(要先裝好unzip)：在下載網址的下面有指令可以直接執行下載跟解壓縮

## 10.1 將NGC CLI安裝檔上傳進去

下載網址：<https://org.ngc.nvidia.com/setup/installers/cli>
LAB中的安裝檔為2025/12/10釋出的，版本為4.10.0
檔名為：ngccli\_linux.zip
```
root@gpuserver:/home/ubuntu/# mkdir ngccli
root@gpuserver:/home/ubuntu/# cd ngccli
root@gpuserver:/home/ubuntu/ngccli# mv ../ngccli_linux.zip ./
root@gpuserver:/home/ubuntu/ngccli# ls
```

## 10.2 解壓縮NGC CLI安裝檔

這邊要先安裝unzip套件
```
root@gpuserver:/home/ubuntu/ngccli# apt-get install unzip

root@gpuserver:/home/ubuntu/ngccli# unzip ngccli_linux.zip
```

## 10.3 檢查md5 checksum，避免載到有問題的檔案

```
root@gpuserver:/home/ubuntu/ngccli# find ngc-cli/ -type f -exec md5sum {} + | LC_ALL=C sort | md5sum -c ngc-cli.md5
#下面返回OK即可
-: OK

root@gpuserver:/home/ubuntu/ngccli# sha256sum ngccli_linux.zip
3e1d3ab23e5b4e8ffc704bf1da4c775a1d68d7bdf8f6d7101b4c85da604d1a58  ngccli_linux.zip

官網提供的字串如下，確認相同即可
3e1d3ab23e5b4e8ffc704bf1da4c775a1d68d7bdf8f6d7101b4c85da604d1a58
```

## 10.4 將NGC CLI路徑設定進環境變數PATH中

```
root@gpuserver:/home/ubuntu/ngccli# chmod u+x ngc-cli/ngc
root@gpuserver:/home/ubuntu/ngccli# echo "export PATH=\"\$PATH:$(pwd)/ngc-cli\"" >> ~/.bash_profile && source ~/.bash_profile
```

## 10.5 驗證NGC CLI指令正常

```
root@gpuserver:/home/ubuntu/ngccli# ngc --version
NGC CLI 4.10.0
```

# 11\. 將環境接上NGC

## 11.1 註冊NVIDIA帳號，並在NGC中建立NVIDIA Cloud Account

NGC是NVIDIA GPU Cloud的縮寫，可能因此需要建立NVIDIA Cloud Account才能對他存取
建立帳號後，登入就可以

## 11.2 於NGC中，產生Personal API Key

### 11.2.1 NGC API Key種類

有兩種：<https://docs.nvidia.com/ngc/latest/ngc-catalog-user-guide.html>

* Personal Key：任何使用者都可以建立，每人最多8個
* Service Key：只有NGC org owners and user\_admins可以建立，最多64個

### 11.2.2 產生Personal Key

2025/12紀錄：NGC CLI部分還沒搞定XD  感覺應該是先在NGC建好NVIDIA Cloud Account，再用這個帳號產生API Key -> 在主機中用ngc config set指令輸出API Key串起來 \[柏翰確定是這樣沒錯\]

1. 連線NGC Catalog網站，網址為：<https://catalog.ngc.nvidia.com/>
2. 點擊右上角"Account Settings"
3. 於\[Account Settings\]頁面下拉，點擊\[Keys & Secrets\]\[API Keys\]欄位中的"Generate API Key"
4. 於\[API Keys\]頁面中，點擊"+Generate Personal Key"
5. 於\[Generate Personal Key\]彈出畫面中，依序填入以下資訊
	1. \[Key Details\]
		1. Key Name：cplin-ngctestkey
		2. Expiration：預設為12個月，另有其他選項 - 1hr, 12hr, 24hr, 7d, 14d, 30d, 3m, 6m, 12m, Never Expire, Custom Time
	2. \[Key Permissions\]
		1. Services Includeed：依需求勾選，免費帳號有NGC Catalog、Secrets Manager兩個選項
6. 點擊Generate後，畫面就會出現Key的內容，要保存好

```
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
```

## 11.3 於GPU VM中，透過指令串接個人的NGC API Key

### 11.3.1 讓NGC CLI設定串接API Key

目的：用以搜尋、管理、下載NGC庫的模型或工具
```
root@gputestvm:/home/gpusr01/ngccli# ngc config set
Enter API key [no-apikey]. Choices: [<VALID_APIKEY>, 'no-apikey']: <貼上前依步驟的Personal API Key>
Enter CLI output format type [ascii]. Choices: ['ascii', 'csv', 'json']: json
Enter org [no-org]. Choices: ['0861072560778282']: 0861072560778282
Enter team [no-team]. Choices: ['no-team']: no-team
Enter ace [no-ace]. Choices: ['no-ace']: no-ace

Validating configuration...
Successfully validated configuration.                        -> 完成驗證設定
Saving configuration...
Successfully saved NGC configuration to /root/.ngc/config    -> 成功儲存設定檔
```

### 11.3.2 讓Docker CLI串接API Key

```
root@gputestvm:/home/gpusr01/ngccli# docker login nvcr.io --username '$oauthtoken'
Password: <貼上前依步驟的Personal API Key>
...
WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'.
Configure a credential helper to remove this warning. See
https://docs.docker.com/go/credential-store/

Login Succeeded  -> 成功登入
```
