# K8s 平台建置指引

> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/system-requirements  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix  
> 來源：https://docs.nvidia.com/ai-enterprise/deployment/bare-metal/latest/kubernetes.html  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25  
> 整理日期：2026-05-20  
> 目標版本：Kubernetes 1.33 – 1.35（Run:ai v2.25 相容）

---

## 1. 本案環境概述

| 角色 | 設備 | OS | K8s 角色 |
|------|------|-----|----------|
| 管理節點 1 | Dell R670 #1 | Ubuntu 24.04 LTS | Control Plane (Master) |
| 管理節點 2 | Dell R670 #2 | Ubuntu 24.04 LTS | Control Plane (Master) / Worker |
| GPU 工作節點 | Dell XE9780 | Ubuntu 24.04 LTS（Dell 官方認證版本，Canonical Cert ID: 202510-37982） | Worker Node（GPU） |

> **OS 選型依據**：Dell PowerEdge XE9780 的 Canonical Ubuntu 認證僅涵蓋 Ubuntu 24.04 LTS；GPU Operator 26.3.x 支援 Ubuntu 22.04 / 24.04，本案統一以 **Ubuntu 24.04 LTS** 建置以確保 Dell 官方硬體認證對齊。

**本案 Control Plane 拓撲說明**：本案配置 2 台 R670 作為 control plane 節點。2 個 control plane 可透過 Load Balancer 或 VIP（HAProxy + Keepalived / kube-vip）提供 **API endpoint 備援**，但 **stacked etcd 需至少 3 個 control plane 節點才能形成完整 quorum（奇數票選）**；2 節點 etcd 在一節點故障時將失去 quorum 導致叢集唯讀。本案為已知限制，若需完整 HA，建議補第三台 control plane 節點或改用 external etcd。

### Ubuntu 24.04 LTS 建置注意事項

| 項目 | Ubuntu 22.04 | Ubuntu 24.04（本案） | 說明 |
|------|-------------|---------------------|------|
| 預設核心版本 | 5.15 LTS | 6.8（HWE: 6.11+）| GPU Operator 26.3.x 均支援；建議鎖定核心版本避免自動升級 |
| containerd 套件 | `containerd`（Ubuntu repo）可用 | 建議 `containerd.io`（Docker repo）| Ubuntu 24.04 原生 containerd 版本較舊，Docker repo 版本更新且相容性更佳 |
| `apt-transport-https` | 需安裝 | 過渡套件（可略）| Ubuntu 24.04 中 HTTPS 功能已內建於 `apt`，`ca-certificates` 即可 |
| AppArmor | 選用 | 預設啟用（LSM）| containerd 與 K8s 正常運作，但需注意自訂 Security Profile 若有設定 |

> **核心版本鎖定（所有節點）**：
> ```bash
> # 查詢目前核心版本
> uname -r
> # 鎖定核心套件，防止 apt upgrade 自動升級
> apt-mark hold linux-image-generic linux-headers-generic
> ```

---

## 2. K8s 發行版選型

> ⚠️ **Dell AI Factory orchestration 參考路徑**：Dell 官方 2-8-9-400 brief（XE9680 參考平台）所列的 Kubernetes 發行版為 **Upstream Kubernetes 或 Red Hat OpenShift Container Platform**；本文件採用 RKE2 / kubeadm 屬本案部署選型，均列於 Run:ai 官方支援矩陣。若需完整對齊 Dell AI Factory endorsed stack，請由 Dell / NVIDIA SE 確認適用的發行版。

### 選項 A：RKE2（可選，Ubuntu 24.04 支援；需確認 RKE2/Kubernetes 版本、Ingress Controller 與 Run:ai 相容性）

**特性：**
- 安全性優先設計（符合 CIS benchmark）
- 內建 Containerd（Run:ai 支援的 Container Runtime）
- 列於 NVIDIA AI Enterprise Support Matrix（SUSE Rancher RKE2 1.33–1.35 + Ubuntu 24.04 + containerd 為支援組合）
- SUSE RKE2 Support Matrix 已列 Ubuntu 24.04 ✅

**部署前必確認事項：**

| 確認項目 | 說明 |
|---------|------|
| **RKE2 版本選擇** | 須選擇對應 Kubernetes **1.33–1.35** 的 RKE2 版本（Run:ai v2.25 相容範圍）；建議優先選 **1.34 或 1.35**，避免 1.33 接近 EOL |
| **內建 ingress-nginx EOL** | RKE2 目前內建 `ingress-nginx`，但 **ingress-nginx 上游已於 2026-03 進入 EOL**；RKE2 v1.36 起新叢集預設將轉向 Traefik。在 K8s 1.33–1.35 範圍內仍可使用，但需確認企業資安政策是否接受 EOL 元件，並制定維護策略 |
| **Ingress 替代方案** | 若不接受 ingress-nginx EOL 風險，可改採 **HAProxy Ingress**（Run:ai 官方文件範例），或啟用 Run:ai v2.25 的 **Kubernetes Gateway API（opt-in）**。⚠️ 若採 RKE2 + Gateway API，需搭配 **Traefik ingress-controller** 並確認 Gateway API CRDs 與 Traefik 設定；不可直接沿用 ingress-nginx 路徑 |

**安裝方式（快速參考）：**
```bash
# 指定完整 RKE2 release 版本（格式：v<K8s版本>+rke2r<patch>）
# 請至 https://github.com/rancher/rke2/releases 確認當前最新版本再填入，不可使用 x 佔位符
# 範例（以撰文時 v1.35 最新版為例，部署前請確認是否有更新的 patch）：
INSTALL_RKE2_VERSION="v1.35.4+rke2r1"   # ← 部署前至 releases 頁面確認最新版號

# Master 節點
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=${INSTALL_RKE2_VERSION} sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service

# 取得 Join Token
cat /var/lib/rancher/rke2/server/node-token

# Worker 節點加入
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_VERSION=${INSTALL_RKE2_VERSION} sh -
# 設定 /etc/rancher/rke2/config.yaml 填入 server 與 token
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```

**RKE2 kubeconfig 路徑：**
```bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
# 或複製至 ~/.kube/config
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
```

> **參考來源**：[RKE2 官方文件](https://docs.rke2.io/) ｜ [SUSE RKE2 Support Matrix](https://www.suse.com/suse-rke2/support-matrix/all-supported-versions/) ｜ [NVIDIA AI Enterprise Support Matrix](https://docs.nvidia.com/ai-enterprise/latest/support-matrix/index.html)

---

### 選項 B：kubeadm（Vanilla Kubernetes）

**優點：**
- 最接近標準 Kubernetes，社群資源豐富
- 彈性高，可自由選擇各元件版本

**缺點：**
- 需手動安裝 Ingress Controller、Load Balancer
- HA 配置需額外配置 etcd 叢集或堆疊式 etcd

**安裝方式（快速參考）：**
```bash
# 所有節點：關閉 Swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# 所有節點：安裝 containerd.io（Ubuntu 24.04 建議使用 Docker 官方 repo，避免與原生套件版本差異）
apt-get update && apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  tee /etc/apt/sources.list.d/docker.list
apt-get update && apt-get install -y containerd.io

# 設定 containerd config 並啟用 SystemdCgroup（Kubernetes 必要）
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# 所有節點：安裝 kubeadm, kubelet, kubectl
# Ubuntu 24.04 中 apt-transport-https 為過渡套件，ca-certificates 已涵蓋其功能
# 建議選擇 Kubernetes 1.34 或 1.35（Run:ai v2.25 支援範圍，1.33 接近 EOL）
K8S_MINOR="1.34"   # ← 可改為 1.35；確認最新 patch 版本請見 https://kubernetes.io/releases/
apt-get update && apt-get install -y ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v${K8S_MINOR}/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${K8S_MINOR}/deb/ /" | \
  tee /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# Master 節點：初始化叢集
kubeadm init --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint="<VIP 或 FQDN>:6443"

# 設定 kubeconfig
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# 安裝 CNI（以 Flannel 為例）
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Worker 節點加入（使用 kubeadm init 輸出的 join 指令）
kubeadm join <VIP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

> **部署前請至 [Kubernetes 官方文件](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) 確認 v1.34 / v1.35 最新安裝步驟**

---

## 3. 必要元件安裝（kubeadm 環境）

RKE2 環境已內建部分元件，kubeadm 環境需手動安裝以下項目：

### 3.1 CNI 網路插件

選擇其一安裝：

| CNI | 特點 | 安裝複雜度 |
|-----|------|-----------|
| Flannel | 簡單，適合測試 | 低 |
| Calico | 支援 NetworkPolicy，生產推薦 | 中 |
| Cilium | 高效能，支援 eBPF | 高 |

```bash
# Calico 安裝範例（<CALICO_VERSION> 請至 https://github.com/projectcalico/calico/releases 確認最新版本）
# 部署前請至 [Calico 官方安裝文件](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart) 確認步驟
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/<CALICO_VERSION>/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/<CALICO_VERSION>/manifests/custom-resources.yaml
```

### 3.2 Ingress Controller（NGINX）

Run:ai 需要 Ingress Controller，kubeadm 環境需手動安裝：

```bash
# NGINX Ingress Controller（官方 Helm 方式）
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

> Run:ai 安裝時使用 `--set global.ingress.ingressClass=nginx` 指定

> 📌 **v2.25 新功能：Kubernetes Gateway API（opt-in）**  
> Run:ai v2.25 已支援以 **Kubernetes Gateway API** 取代 Ingress，並提供零停機切換能力，但目前仍為 **opt-in（非預設）**。  
> **Ingress 仍是預設選項**，本案建置使用 Ingress 即可；若未來需要切換至 Gateway API，請參閱官方遷移文件。  
> 來源：[What's New v2.25 – Kubernetes Gateway API](https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25)

### 3.3 MetalLB（Bare Metal LoadBalancer）

裸機環境需 MetalLB 提供 LoadBalancer IP 功能：

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace

# 配置 IP 位址池（依環境調整 IP 範圍）
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.20.10.0/24   # 替換為實際可用 IP 範圍
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
EOF
```

### 3.4 Default Storage Class

Control Plane 需要 Default Storage Class，常見選擇：

```bash
# 使用 local-path-provisioner（測試/小型環境）
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

> 生產環境建議使用 NFS、Longhorn 或其他企業儲存方案

### 3.5 Prometheus / Kube-Prometheus Stack

Run:ai 依賴 Prometheus 收集指標，**必須安裝**：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --version 76.0.0    # 請確認符合 Run:ai v2.25 需求（>=76.0）
```

---

## 4. 安裝後驗證

```bash
# 確認所有節點就緒
kubectl get nodes -o wide

# 確認系統 Pod 正常
kubectl get pods -A

# 確認 Ingress Controller 正常（依實際採用方案擇一執行）
# 方案 A：ingress-nginx（kubeadm 手動安裝 或 RKE2 內建，注意 2026-03 EOL）
kubectl get pods -n ingress-nginx
# 方案 B：HAProxy Ingress（Run:ai 官方文件範例）
# kubectl get pods -n haproxy-controller
# 方案 C：RKE2 內建 Traefik（kube-system namespace）
# kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
# 方案 D：Gateway API（確認 GatewayClass 與 Gateway 資源）
# kubectl get gatewayclass && kubectl get gateway -A

# 確認 MetalLB 正常
kubectl get pods -n metallb-system

# 確認 Prometheus 正常
kubectl get pods -n monitoring

# 確認 Default Storage Class
kubectl get storageclass
```

**期望輸出（節點）：**
```
NAME        STATUS   ROLES           AGE   VERSION
r670-01     Ready    control-plane   Xm    v1.34.x   # 或 v1.35.x，依實際選型
r670-02     Ready    control-plane   Xm    v1.34.x
xe9780      Ready    <none>          Xm    v1.34.x
```

---

## 5. NTP 時間同步確認

**必須確保所有節點時間同步**（Run:ai 官方要求）：

Ubuntu 24.04 LTS 預設已啟用 `systemd-timesyncd` 作為 NTP 客戶端，通常無需另外安裝 chrony。

```bash
# 確認 NTP 同步狀態（Ubuntu 24.04 預設使用 systemd-timesyncd）
timedatectl status
timedatectl show-timesync

# 若企業環境需指定內部 NTP Server，編輯設定後重啟服務
# 編輯 /etc/systemd/timesyncd.conf，加入：NTP=<企業NTP伺服器IP>
systemctl restart systemd-timesyncd
timedatectl status

# 若環境政策要求使用 chrony，可額外安裝（會取代 systemd-timesyncd）
# apt-get install -y chrony
# systemctl enable --now chrony
# chronyc tracking
```

---

## 6. 下一步

完成 K8s 叢集建置後，繼續安裝 NVIDIA GPU Operator：

→ [03-NVIDIA-GPU-Operator-Setup.md](03-NVIDIA-GPU-Operator-Setup.md)
