# 01 – RKE2 部署前置需求（Ubuntu 24.04 LTS）

> 來源：
> - [https://docs.rke2.io/install/requirements](https://docs.rke2.io/install/requirements)
> - [https://docs.rke2.io/known_issues](https://docs.rke2.io/known_issues)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. 作業系統支援

| 項目 | 說明 |
|------|------|
| 官方支援聲明 | "RKE2 should work on any Linux distribution that uses systemd and iptables"（[來源](https://docs.rke2.io/install/requirements)） |
| 架構 | x86_64、arm64/aarch64 |
| Ubuntu 24.04 LTS 狀態 | **未在官方文件中明確列入「已驗證」清單**，但滿足 systemd + iptables 要求，廣泛被社群採用。**待驗證**：建議在 PoC 階段驗證關鍵功能後再進入生產 |

> **重要**：SUSE 官方的 Support Matrix（[https://www.suse.com/suse-rke2/support-matrix/](https://www.suse.com/suse-rke2/support-matrix/)）以 SLES、RHEL、Rocky 等為主要驗證對象。如組織需要原廠付費支援（subscription），請以該 Support Matrix 列出的 OS 為準。

---

## 2. 硬體規格

### 2.1 最低需求（來源：[install/requirements](https://docs.rke2.io/install/requirements)）

| 項目 | 最低 | 建議 |
|------|------|------|
| RAM | 4 GB | 至少 8 GB |
| CPU | 2 cores | 至少 4 cores |
| 磁碟 | 無明確最低值 | **SSD**（RKE2 效能取決於 etcd 效能） |

### 2.2 依 Agent 節點數量的 Server 規模建議

| CPU | RAM | 可支援 Agent 節點數 |
|-----|-----|---------------------|
| 2 | 4 GB | 0 – 225 |
| 4 | 8 GB | 226 – 450 |
| 8 | 16 GB | 451 – 1300 |
| 16+ | 32 GB | 1300+ |

### 2.3 本案實際規格建議

| 角色 | 數量 | vCPU | RAM | 磁碟 | 平台 |
|------|------|------|-----|------|------|
| Server | 3 | 8 | 16 GB | 100 GB SSD（**etcd 需要低延遲**） | VMware VM |
| CPU Worker | 2 | 8 | 32 GB | 200 GB | VMware VM |
| GPU Worker（HGX） | 1 | 由實體機決定 | 由實體機決定（≥ 256 GB 建議） | ≥ 500 GB NVMe | Bare-metal Ubuntu |

> **VMware 注意**：將 etcd 對應的 VMDK 放在低延遲儲存（NVMe / 高效能 vSAN tier）。etcd 對磁碟 fsync 延遲敏感（< 10 ms 為佳）。**待驗證**：實際部署前以 `etcdctl check perf` 驗證。

---

## 3. 網路埠需求

來源：[https://docs.rke2.io/install/requirements](https://docs.rke2.io/install/requirements)

### 3.1 核心 Inbound 規則

| 埠 | 協定 | 來源 | 目的地 | 用途 |
|----|------|------|--------|------|
| 6443 | TCP | 所有節點、kubectl 客戶端 | Server | Kubernetes API |
| 9345 | TCP | 所有節點 | Server | RKE2 supervisor API（節點註冊） |
| 10250 | TCP | 所有節點 | 所有節點 | kubelet metrics |

### 3.2 etcd（Server ↔ Server）

| 埠 | 協定 | 用途 |
|----|------|------|
| 2379 | TCP | etcd client |
| 2380 | TCP | etcd peer |
| 2381 | TCP | etcd metrics |

### 3.3 CNI 特定埠（預設 Canal = Calico + Flannel）

| 埠 | 協定 | 用途 |
|----|------|------|
| 8472 | UDP | Flannel VXLAN（**不可暴露到公網**） |
| 9099 | TCP | Canal 健康檢查 |
| 179 | TCP | Calico BGP（僅 Calico 模式需要） |
| 4789 | UDP | Windows 節點 Calico/Flannel VXLAN |

### 3.4 NodePort

| 埠 | 協定 | 用途 |
|----|------|------|
| 30000 – 32767 | TCP | Service NodePort 範圍（預設） |

---

## 4. Ubuntu 24.04 前置設定

### 4.1 必要套件

```bash
sudo apt-get update
sudo apt-get install -y curl iptables sudo systemd
```

> **NetworkManager**：Ubuntu Server 24.04 預設使用 systemd-networkd 或 netplan，不一定安裝 NetworkManager。若有安裝（如部分桌面或客製映像），需依 [§5.2](#52-networkmanager) 設定。

### 4.2 防火牆（ufw）

來源：[https://docs.rke2.io/known_issues](https://docs.rke2.io/known_issues)

> "Firewalld conflicts with RKE2's default Canal networking stack."

Ubuntu 預設防火牆為 **ufw**（非 firewalld），但同樣可能干擾 Canal。建議：

```bash
# 方案 A：直接停用（最簡單，叢集內部信任）
sudo ufw disable

# 方案 B：保留 ufw 但開放叢集間流量（依組織安全政策）
sudo ufw allow from <叢集節點 CIDR>
```

### 4.3 Kernel 模組與 sysctl

RKE2 啟動時會嘗試載入所需模組。可預先確認：

```bash
sudo modprobe br_netfilter
sudo modprobe overlay
echo -e "br_netfilter\noverlay" | sudo tee /etc/modules-load.d/rke2.conf
```

建議的 sysctl（**註**：以下為 Kubernetes 社群通用建議；RKE2 在 CIS 模式下會另外提供 `/usr/share/rke2/rke2-cis-sysctl.conf`，詳見 [§6](#6-cis-合規模式選用)）：

```bash
cat <<EOF | sudo tee /etc/sysctl.d/90-rke2.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sudo sysctl --system
```

### 4.4 主機名與 DNS

> "Node names must be unique"（[來源](https://docs.rke2.io/install/requirements)）

- 每節點 `hostname` 必須唯一。
- 內部 DNS 解析必須正確：每節點能解析其他節點 hostname → IP。
- 建議於 `/etc/hosts` 預先寫入所有節點對應，避免依賴 DNS 啟動順序。

### 4.5 時間同步

雖未在官方需求中明列，但 etcd 對時間漂移敏感。Ubuntu 24.04 預設啟用 `systemd-timesyncd`，確認狀態：

```bash
timedatectl status
```

### 4.6 Swap

Kubernetes 建議關閉 swap：

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 4.7 AppArmor

Ubuntu 24.04 預設啟用 AppArmor。RKE2 / containerd 與 AppArmor 通常相容；**待驗證**：若部署 Run:ai 後 GPU 容器出現權限錯誤，可暫時將 nvidia-container-toolkit 相關進程 profile 設為 complain 模式排查。

---

## 5. 已知問題與環境設定

### 5.1 防火牆（firewalld / ufw）

見 [§4.2](#42-防火牆ufw)。

### 5.2 NetworkManager

來源：[https://docs.rke2.io/known_issues](https://docs.rke2.io/known_issues)

若主機安裝 NetworkManager，必須避免其管理 CNI 介面：

```bash
sudo tee /etc/NetworkManager/conf.d/rke2-canal.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
EOF
sudo systemctl reload NetworkManager
```

### 5.3 Calico VXLAN Checksum

> "Kernel bug affects Calico with VXLAN encapsulation on Ubuntu 18.04/20.04 and openSUSE Leap 15.3"

Ubuntu 24.04 不在此 bug 影響範圍，但仍建議測試 Pod 跨節點通訊以確認。

### 5.4 NetworkManager 衍生服務（RHEL 系）

RHEL 8.4 需停用 `nm-cloud-setup.service`，**Ubuntu 24.04 不適用此項**。

### 5.5 Pod IP 耗盡

來源：[https://docs.rke2.io/known_issues](https://docs.rke2.io/known_issues)

兩個常見成因：
1. iptables 二進位缺失 → Pod 建立失敗 → `apt-get install iptables`。
2. `/var/lib/cni/networks/k8s-pod-network` 殘留 lock 檔 → drain 並重啟節點。

---

## 6. CIS 合規模式（選用）

來源：[https://docs.rke2.io/security/hardening_guide](https://docs.rke2.io/security/hardening_guide)

若有合規需求，於 `/etc/rancher/rke2/config.yaml` 加入：

```yaml
profile: "cis"
```

額外要求：

1. **套用 RKE2 提供的 sysctl**：
   ```bash
   sudo cp -f /usr/share/rke2/rke2-cis-sysctl.conf /etc/sysctl.d/60-rke2-cis.conf
   sudo systemctl restart systemd-sysctl
   ```
2. **建立 etcd 系統使用者**（CIS profile 要求 etcd data dir owner）：
   ```bash
   sudo useradd -r -c "etcd user" -s /sbin/nologin -M etcd -U
   ```
3. CIS 模式會啟用 **Pod Security Admission（restricted）** 與 **預設 NetworkPolicy**；應用部署需自行調整對應的安全標籤。

> 本案若未明確要求 CIS 合規，可暫不啟用，待維運穩定後再切換。

---

## 7. VMware VM 特有注意事項

| 項目 | 建議 |
|------|------|
| VM Hardware Version | 19 以上 |
| VMware Tools / open-vm-tools | 安裝 `open-vm-tools`（`sudo apt-get install -y open-vm-tools`） |
| CPU/MMU 虛擬化 | 開啟（預設） |
| Memory Reservation | Server VM 建議全保留以避免 etcd 因 ballooning 影響 |
| Disk Provisioning | **Thick eager-zeroed** 用於 etcd 磁碟 |
| NIC | VMXNET3 |
| DRS / vMotion | Server 節點建議設定 DRS Affinity Rule，**避免 3 個 Server VM 同時在同一台 ESXi 主機**（單點故障風險） |

---

## 8. GPU Worker（HGX 實體機）特有需求

| 項目 | 說明 |
|------|------|
| Ubuntu 版本 | 24.04 LTS |
| NVIDIA Driver | **不需手動安裝**，由 GPU Operator 部署；但若已預裝，須與 GPU Operator 版本相容。詳見 [03-RKE2-Worker-Node-Setup.md](03-RKE2-Worker-Node-Setup.md) |
| `nouveau` 模組 | 確認停用（GPU Operator 會處理，但預先 blacklist 可避免初次啟動衝突） |
| BIOS | 啟用 SR-IOV（**若使用 vGPU**，本案非 vGPU 不需要）、IOMMU、確認 NUMA 拓撲已暴露 |
| InfiniBand / NVLink | 若有 IB 網路給 Run:ai distributed training，需安裝對應驅動（Mellanox OFED / NVIDIA DOCA-OFED）。進階 RDMA / Multi-Node NVLink / GB200 場景另需 NVIDIA Network Operator，詳見下方說明 |

驗證 nouveau 已停用：

```bash
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
echo "options nouveau modeset=0" | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
sudo reboot
lsmod | grep nouveau   # 應無輸出
```

### 8.1 RDMA / InfiniBand / Multi-Node NVLink 進階網路需求（Run:ai v2.25）

> 以下版本範圍以 [Run:ai Self-hosted v2.25 support matrix](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/support-matrix.md) 為準；部署前請至該頁面確認最新相容性。

| 元件 | Run:ai 2.25 支援範圍 | 適用場景 |
|------|---------------------|----------|
| **NVIDIA Network Operator** | **25.10 – 26.1** | RDMA、InfiniBand（IB）、Multi-Node NVLink（GB200 NVL 平台） |
| **DRA Driver（GPU / GDR）** | **25.8 – 25.12** | GPU 直接記憶體存取加速（GPUDirect RDMA）、Multi-Node NVLink |

**部署說明：**

1. **NVIDIA Network Operator** 負責管理 IB 網卡驅動（MLNX_OFED / DOCA-OFED）、SR-IOV Device Plugin、RDMA 相關 ConfigMap；在 Run:ai 分散式 training 場景中，GPU 節點需透過 IB 介面通訊時為必要元件。
2. **DRA（Dynamic Resource Allocation）Driver** 用於 GB200 NVL 或 GPUDirect RDMA 加速場景，需 Kubernetes 1.32+（beta API）。RKE2 v1.35 對應 K8s 1.35，符合此版本需求。**但 NVIDIA DRA Driver 並非 RKE2 內建元件**，仍需依 [NVIDIA / Run:ai 官方文件](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/install-using-helm/system-requirements.md) 另行部署。
3. 本案 HGX XE9780 **若僅使用乙太網路（無 IB 網卡）**，可暫不安裝 Network Operator；但若日後計畫導入 IB 或多機 NVLink 訓練，請提前規劃。
4. Network Operator 部署方式同 GPU Operator，透過 RKE2 HelmChart CR；詳細步驟請參考 [NVIDIA Network Operator 官方文件](https://docs.nvidia.com/networking/software/cloud-orchestration/network-operator/)。

---

## 9. 部署前檢查清單

- [ ] 所有節點主機名唯一
- [ ] 所有節點 `/etc/hosts` 預寫對應或內部 DNS 正常
- [ ] 所有節點時間同步（`timedatectl status`）
- [ ] 所有節點 swap 已關閉
- [ ] 所有節點防火牆已配置（停用或開放叢集 CIDR）
- [ ] 所有節點可互通埠 6443、9345、10250、2379-2381、8472/UDP
- [ ] 已決定 RKE2 channel（建議 `stable`，對應 K8s 1.35 以符合 Run:ai 2.25）
- [ ] 已決定 HA 註冊地址方案（L4 LB / DNS Round-robin / VIP）詳見 [02-RKE2-Installation-HA.md](02-RKE2-Installation-HA.md)
- [ ] GPU 節點 nouveau 已停用
