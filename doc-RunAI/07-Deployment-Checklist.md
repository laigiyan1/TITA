# 逐步部署清單

> 整理日期：2026-05-20  
> 目標版本：NVIDIA Run:ai v2.25 + Kubernetes 1.33–1.35 + GPU Operator 26.3.x（建議；25.10.x 已列為 Deprecated）  
> 本案硬體：Dell XE9780（GPU）× 1 + Dell R670（管理節點）× 2

> ⚠️ **Dell AI Factory 參考配置與本案差異說明**  
> Dell 官方 2-8-9-400 brief 的參考平台為 **Dell PowerEdge XE9680**（GPU worker nodes，up to 12 台）+ **Dell R670**（管理節點）× 4 + Spectrum-4 SN5610 網路交換器 + Dell PowerScale F710（NFS 共享儲存）+ Upstream Kubernetes 或 Red Hat OpenShift Container Platform。  
> 本案為**縮減部署**：XE9780（GPU 型號待 BOM 確認）× 1 + R670 × 2，Kubernetes 發行版採 RKE2 / kubeadm（均列於 Run:ai 官方支援矩陣），儲存 / 網路設備依採購 BOM 確認。  
> XE9780 的 Dell AI Factory endorsement 狀態需由 Dell / NVIDIA SE 確認（Dell 官方 brief 明確列出的參考平台為 XE9680）。

---

## 使用方式

依序執行下列步驟，完成後在 `[ ]` 處標記 `[x]`。  
每個步驟完成後執行對應的「驗證指令」確認正確後再繼續。

---

## 第一階段：硬體與 OS 準備

### A. 所有節點（R670 × 2 + XE9780 × 1）

- [ ] **A1** 安裝 Ubuntu 22.04 LTS
- [ ] **A2** 更新系統套件
  ```bash
  apt-get update && apt-get upgrade -y
  ```
- [ ] **A3** 設定固定 IP 位址與 Hostname
  ```bash
  hostnamectl set-hostname <node-name>
  ```
- [ ] **A4** 設定 `/etc/hosts`（所有節點互相加入）
  ```
  <R670-1-IP>   r670-01.example.internal r670-01
  <R670-2-IP>   r670-02.example.internal r670-02
  <XE9780-IP>   xe9780.example.internal  xe9780
  ```
- [ ] **A5** 設定 NTP 時間同步
  ```bash
  apt-get install -y chrony
  systemctl enable --now chrony
  chronyc tracking   # 確認 Stratum ≠ 16
  ```
- [ ] **A6** 關閉 Swap（K8s 必要）
  ```bash
  swapoff -a
  sed -i '/swap/d' /etc/fstab
  ```
- [ ] **A7** 設定核心模組與系統參數
  ```bash
  cat <<EOF | tee /etc/modules-load.d/k8s.conf
  overlay
  br_netfilter
  EOF
  modprobe overlay && modprobe br_netfilter
  
  cat <<EOF | tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF
  sysctl --system
  ```
- [ ] **A8** 安裝 Containerd
  ```bash
  apt-get install -y containerd
  containerd config default | tee /etc/containerd/config.toml
  sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
  systemctl restart containerd
  ```

### B. GPU 節點專屬（XE9780 only）

- [ ] **B1** 確認 GPU 型號並對照 [GPU Operator 支援清單](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html)
- [ ] **B2** 停用自動核心更新
  ```bash
  apt-mark hold linux-image-generic linux-headers-generic
  ```
- [ ] **B3** 確認 GPU 在 OS 層級可見
  ```bash
  lspci | grep -i nvidia
  ```

---

## 第二階段：Kubernetes 平台建置

> 選擇 kubeadm（選項 B）或 RKE2（選項 A）其中一種

### C. RKE2 安裝（選項 A，建議）

- [ ] **C1** 在 R670-01（第一個 Master）安裝 RKE2 Server
  ```bash
  curl -sfL https://get.rke2.io | sh -
  systemctl enable rke2-server
  systemctl start rke2-server
  ```
- [ ] **C2** 取得 Join Token
  ```bash
  cat /var/lib/rancher/rke2/server/node-token
  ```
- [ ] **C3** 在 R670-02 安裝 RKE2 Server（HA）
- [ ] **C4** 在 XE9780 安裝 RKE2 Agent
- [ ] **C5** 設定 kubeconfig
  ```bash
  mkdir -p ~/.kube
  cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
  export KUBECONFIG=~/.kube/config
  ```

### D. kubeadm 安裝（選項 B）

- [ ] **D1** 安裝 kubeadm、kubelet、kubectl（所有節點）
- [ ] **D2** Master 節點初始化叢集
  ```bash
  kubeadm init --pod-network-cidr=10.244.0.0/16
  ```
- [ ] **D3** 設定 kubeconfig
- [ ] **D4** 安裝 CNI（Calico 或 Flannel）
- [ ] **D5** Worker 節點加入叢集

### E. K8s 叢集驗證

- [ ] **E1** 確認所有節點 Ready
  ```bash
  kubectl get nodes   # 期望：所有節點 STATUS=Ready
  ```

---

## 第三階段：K8s 基礎元件安裝

### F. 安裝 MetalLB（Bare Metal LoadBalancer）

- [ ] **F1** 安裝 MetalLB
  ```bash
  helm repo add metallb https://metallb.github.io/metallb && helm repo update
  helm install metallb metallb/metallb -n metallb-system --create-namespace
  ```
- [ ] **F2** 設定 IP 位址池（填入實際可用 IP 範圍）
- [ ] **F3** 驗證 MetalLB Pod Running
  ```bash
  kubectl get pods -n metallb-system
  ```

### G. 安裝 Ingress Controller（kubeadm 環境需執行；RKE2 跳過）

- [ ] **G1** 安裝 NGINX Ingress Controller
  ```bash
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm repo update
  helm install ingress-nginx ingress-nginx/ingress-nginx \
    -n ingress-nginx --create-namespace \
    --set controller.service.type=LoadBalancer
  ```
- [ ] **G2** 驗證 Ingress Controller Pod Running 且有外部 IP
  ```bash
  kubectl get svc -n ingress-nginx
  ```

### H. 設定 Storage Class

- [ ] **H1** 確認或安裝 Default Storage Class
  ```bash
  kubectl get storageclass | grep default
  ```
- [ ] **H2** 若無 Default Storage Class，安裝 local-path-provisioner
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
  kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
  ```

### I. 安裝 Prometheus

- [ ] **I1** 安裝 Kube-Prometheus Stack
  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
  helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
    -n monitoring --create-namespace
  ```
- [ ] **I2** 確認 Prometheus Pod Running
  ```bash
  kubectl get pods -n monitoring
  ```

---

## 第四階段：NVIDIA GPU Operator

- [ ] **J1** 新增 NVIDIA Helm Repo
  ```bash
  helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update
  ```
- [ ] **J2** 建立命名空間
  ```bash
  kubectl create ns gpu-operator
  kubectl label --overwrite ns gpu-operator pod-security.kubernetes.io/enforce=privileged
  ```
- [ ] **J3** 安裝 GPU Operator（建議選用 26.3.x；25.10.x 雖在 Run:ai 支援範圍內但已列為 Deprecated）
  > **版本選擇**：Run:ai v2.25 支援範圍 25.10–26.3；[GPU Operator 生命週期政策](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html)顯示 **25.10.x = Deprecated、26.3.x = Current**，**建議選用 26.3.x**。  
  > 部署前至 [Run:ai Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix) 交叉確認。截至 2026-05 已知：v26.3.1 在支援範圍內且為 Current。
  ```bash
  GPU_OPERATOR_VERSION="v26.3.1"   # 部署前確認為最新相容版本
  helm install --wait --generate-name \
    -n gpu-operator --create-namespace \
    nvidia/gpu-operator --version=${GPU_OPERATOR_VERSION}
  ```
- [ ] **J4** 驗證 GPU Operator Pod 狀態（所有 Pod Running 或 Completed）
  ```bash
  kubectl get pods -n gpu-operator
  ```
- [ ] **J5** 確認 GPU 資源可見
  ```bash
  kubectl describe node xe9780 | grep "nvidia.com/gpu"
  ```
- [ ] **J6** 執行 CUDA 測試工作負載，確認 GPU 可用（期望：`Test PASSED`）

---

## 第五階段：DNS 與 TLS 憑證

> ⚠️ **v2.25 變更：Host-based Routing 預設啟用（Kubernetes 環境）**  
> 本案採 Kubernetes + 預設 Host-based Routing，萬用字元 DNS 與 TLS 為必要；若改用 path-based routing 則不需要萬用字元憑證；OpenShift 環境另有不同要求。  
> 來源：[What's New v2.25](https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25) | [Cluster System Requirements](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/system-requirements)

- [ ] **K1** 設定 DNS A Record：`runai.<DOMAIN>` → Ingress Controller External IP
- [ ] **K2** 設定**萬用字元** DNS A Record：`*.<DOMAIN>` → Ingress Controller External IP（本案 Kubernetes + 預設 Host-based Routing 必要）
- [ ] **K3** 確認 DNS 解析正常（含萬用字元解析）
  ```bash
  nslookup runai.<DOMAIN>
  nslookup test.runai.<DOMAIN>   # 萬用字元子網域測試
  ```
- [ ] **K4** 準備 TLS 憑證：
  - `runai.<DOMAIN>` 憑證（Control Plane 主網域）
  - `*.<DOMAIN>` 萬用字元憑證（Kubernetes + 預設 Host-based Routing 時必要；改用 path-based routing 時不需要）
  - `*.<inference-domain>` 萬用字元憑證（Inference，選用）

---

## 第六階段：Run:ai Control Plane 安裝

- [ ] **L1** 建立 NGC API Key（https://ngc.nvidia.com/）
- [ ] **L2** 新增 Run:ai Helm Repo
  ```bash
  helm repo add runai https://helm.ngc.nvidia.com/nvidia/runai \
    --force-update \
    --username='$oauthtoken' \
    --password=<NGC_API_KEY>
  helm repo update
  ```
- [ ] **L3** 建立 runai-backend 命名空間與映像拉取 Secret
- [ ] **L4** 安裝 Control Plane
  ```bash
  helm upgrade -i runai-backend -n runai-backend runai/control-plane \
    --set global.domain=<DOMAIN> \
    --set global.ingress.ingressClass=nginx \
    --set tenantsManager.config.adminUsername=<ADMIN_EMAIL> \
    --set tenantsManager.config.adminPassword="<ADMIN_PASSWORD>"
  ```
- [ ] **L5** 等待所有 Control Plane Pod Running（最多 15 分鐘）
  ```bash
  kubectl get pods -n runai-backend -w
  ```
- [ ] **L6** 瀏覽器登入 `https://runai.<DOMAIN>`，確認 UI 可存取
- [ ] **L7** 修改初始管理員密碼

---

## 第七階段：Run:ai Cluster 安裝

- [ ] **M1** 登入 Run:ai UI，完成 Onboarding Wizard
  - 設定 Cluster 名稱
  - 複製 Wizard 生成的 Helm 指令
- [ ] **M2** 建立 runai 命名空間映像拉取 Secret
- [ ] **M3** 執行 Wizard 生成的 Helm 安裝指令
- [ ] **M4** 監控安裝進度
  ```bash
  kubectl get pods -n runai -w
  ```
- [ ] **M5** 確認 Run:ai UI 顯示「Cluster connected」

---

## 第八階段：安裝後設定

- [ ] **N1** 建立 Department（部門）
- [ ] **N2** 建立 Project（專案），設定 GPU 配額
- [ ] **N3** 建立用戶帳號，分配 Project 存取權限
- [ ] **N4** 安裝 runai CLI（研究員工作站）
- [ ] **N5** 執行 CLI 測試工作負載，確認端到端功能正常

---

## 第九階段：最終驗證

- [ ] **O1** 所有 K8s 節點 Ready
- [ ] **O2** 所有 `runai` 命名空間 Pod Running
- [ ] **O3** 所有 `runai-backend` 命名空間 Pod Running
- [ ] **O4** GPU Operator Pod 正常
- [ ] **O5** Prometheus 正常收集指標
- [ ] **O6** Run:ai 新版 Overview / Analytics 監控頁面可存取（透過 Run:ai UI；v2.25 已移除 legacy Grafana dashboards）
- [ ] **O7** 提交 GPU 工作負載測試成功
- [ ] **O8** Run:ai UI 顯示 GPU 使用率資料

---

## 預估時程參考

| 階段 | 預估時間 |
|------|----------|
| OS 安裝與設定（3 台） | 2–4 小時 |
| K8s 平台建置 | 1–2 小時 |
| K8s 基礎元件安裝 | 1 小時 |
| GPU Operator 安裝 | 30–60 分鐘 |
| Run:ai 安裝 | 30–60 分鐘 |
| 設定與驗證 | 1–2 小時 |
| **總計** | **6–11 小時** |

> 以上時程為估算，實際時間依環境複雜度、網路速度與問題排解狀況而異。
