# 10 – 部署檢查清單

> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

本清單彙整 [01](01-RKE2-Prerequisites.md) – [09](09-RKE2-Troubleshooting.md) 中分散的檢查項目，作為部署前 → 部署中 → 部署後 → 維運週期的單一檢核入口。

---

## A. 規劃階段

- [ ] 已確認 RKE2 版本（建議 `stable` channel → v1.35.5+rke2r1，對應 K8s 1.35，符合 Run:ai 2.25 支援範圍）
- [ ] 已確認 NVIDIA GPU Operator 版本（建議 v25.10.x）
- [ ] 已確認 Ubuntu 版本（Ubuntu 24.04 LTS Server）
- [ ] 已確認節點清單與 IP 規劃（3 server + 2 cpu worker + 1 gpu worker）
- [ ] 已確認 LB 方案（HAProxy / NSX-T LB / keepalived VIP）
- [ ] 已決定固定註冊 FQDN（例：`rke2.example.local`）
- [ ] 已決定 cluster-cidr / service-cidr（建議用預設 10.42/10.43）
- [ ] 已決定 token 內容並安全保存

---

## B. OS 前置（每個節點）

- [ ] Hostname 唯一
- [ ] `/etc/hosts` 已寫入全部節點對應
- [ ] 時間同步 OK（`timedatectl status`）
- [ ] Swap 已關閉
- [ ] 防火牆已配置（停用 ufw 或開放叢集 CIDR）
- [ ] `iptables` 套件已安裝
- [ ] `br_netfilter`、`overlay` kernel 模組可載入
- [ ] sysctl 已套用（`net.ipv4.ip_forward=1` 等）
- [ ] 必要埠（6443、9345、10250、2379-2381、8472/UDP）可互通
- [ ] open-vm-tools 已安裝（VMware VM）
- [ ] nouveau 已停用（GPU 節點）

---

## C. LB / 註冊地址

- [ ] L4 LB（6443、9345）已部署並通過健康檢查
- [ ] DNS：`rke2.example.local` 已指向 LB VIP
- [ ] 從每個節點 `nc -zv rke2.example.local 9345` 通

---

## D. Server 部署

### Server #1（rke2-srv-01）

- [ ] `/etc/rancher/rke2/config.yaml` 寫入（token、tls-san、write-kubeconfig-mode）
- [ ] RKE2 安裝（`INSTALL_RKE2_CHANNEL=stable`）
- [ ] systemd enable + start
- [ ] `kubectl get nodes` 顯示 srv-01 Ready
- [ ] `kubectl get pods -A` 系統 pod 全部 Running

### Server #2、#3

- [ ] config 含 `server: https://rke2.example.local:9345`、`token` 相同、`tls-san` 一致
- [ ] 等 srv-01 完整就緒後再啟動
- [ ] 加入後 `kubectl get nodes` 顯示 3 個 server
- [ ] etcd `endpoint health --cluster` 3 個 healthy

---

## E. Agent 部署

### CPU Worker × 2

- [ ] config 含 `server`、`token`、`node-label: workload=cpu`
- [ ] RKE2 安裝（`INSTALL_RKE2_TYPE=agent`）
- [ ] 加入後 `kubectl get nodes` 顯示 Ready
- [ ] 加上 `node-role.kubernetes.io/worker=true` 標籤

### GPU Worker × 1

- [ ] config 含 `server`、`token`、`node-label: workload=gpu`、`node-taint: nvidia.com/gpu=true:NoSchedule`
- [ ] RKE2 agent 安裝
- [ ] 節點 Ready
- [ ] GPU Operator HelmChart 已部署
- [ ] `nvidia.com/gpu` allocatable > 0
- [ ] 測試 Pod（nbody）成功執行

---

## F. 驗證

- [ ] 6 節點全部 Ready
- [ ] 系統 namespace 無 CrashLoopBackOff
- [ ] etcd 3 成員 healthy
- [ ] Pod 跨節點通訊正常
- [ ] CoreDNS 解析正常
- [ ] Ingress 測試應用可外部 curl
- [ ] HA failover 測試通過（關閉 1 個 server，API 仍可用）

---

## G. 維運準備

### 備份

- [ ] etcd 自動快照排程已配置（每 6 小時、保留 14 份、啟用壓縮）
- [ ] etcd S3 備份已配置並驗證
- [ ] Token 每日備份至異地
- [ ] 已執行一次完整還原演練

### 監控

- [ ] Prometheus 已部署（Run:ai 前置必要）
- [ ] DCGM Exporter ServiceMonitor 已配置
- [ ] 基本告警（Node NotReady、etcd unhealthy、Pod CrashLoop、snapshot age）已配置
- [ ] 日誌持久化（journald `Storage=persistent`）

### 升級

- [ ] 已確認升級流程文件可被維運人員執行（演練過 1 次）
- [ ] 已備妥 `pre-upgrade` 快照與 rollback SOP

### 文件

- [ ] 此文件庫 ([README](README.md)) 已交付給維運單位
- [ ] 拓撲圖、IP 清單、token 保管位置已記錄於組織 wiki
- [ ] On-call 流程已涵蓋 RKE2 / GPU / Run:ai

---

## H. 上線前最後一哩

- [ ] 與 Run:ai 部署團隊確認 namespace `runai` 已建立
- [ ] 確認 ingress class 名稱與 Run:ai 預期一致
- [ ] **Run:ai 部署模式確認（Ingress / FQDN / TLS）**：
  - **Self-hosted 同叢集**（control plane 與 cluster 安裝於同一個 K8s）：cluster 端**不需另備**獨立 FQDN 與 Ingress controller；確認 control plane 端既有 FQDN / TLS / Ingress 設定可用
  - **Self-hosted 分離叢集**（control plane 與 cluster 各為獨立 K8s）：control plane 端依 [Run:ai Self-hosted control plane system requirements](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/cp-system-requirements) 準備 Ingress / FQDN / TLS；cluster 端依 [Run:ai Self-hosted v2.25 cluster system requirements](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/install-using-helm/system-requirements.md) 準備 Ingress / FQDN / TLS
  - **SaaS**：控制平面由 NVIDIA Run:ai 管理；本地 GPU cluster 仍需依 [Run:ai SaaS system requirements](https://run-ai-docs.nvidia.com/saas/getting-started/installation/install-using-helm/system-requirements) 準備 cluster-side Ingress controller、FQDN 與 TLS certificate，並確認 outbound connectivity 至 Run:ai SaaS 端點；若啟用 inference serving，另需準備 wildcard FQDN 與 wildcard TLS certificate
- [ ] 確認共享儲存（NFS / NAS）已掛載於所有相關節點
- [ ] 已通知資安團隊（防火牆規則、CIS 模式決策）
- [ ] 已通知資料中心團隊（HGX 電力 / 散熱 / 機架）
- [ ] **Kubernetes LoadBalancer 能力確認**：Run:ai 部分元件需要 `LoadBalancer` 類型 Service（取得外部 IP）；on-prem 環境需部署 MetalLB 或使用組織現有 LB 解決方案（NSX-T / F5）提供 VIP 分配
  - MetalLB 部署（與 Run:ai v2.25 system requirements 一致，採 Helm 方式）：
    ```bash
    helm repo add metallb https://metallb.github.io/metallb
    helm install metallb metallb/metallb --version 0.15.3 \
      --namespace metallb-system --create-namespace
    ```
  - 部署後需建立 `IPAddressPool` 與 `L2Advertisement` CR，規劃一段可供分配的 VIP IP pool（與叢集節點 IP 同網段或可路由）
