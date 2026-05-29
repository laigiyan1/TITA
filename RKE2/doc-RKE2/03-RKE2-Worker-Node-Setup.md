# 03 – Worker 節點加入（VM Worker + GPU Bare-metal Worker）

> 來源：
> - [https://docs.rke2.io/install/quickstart](https://docs.rke2.io/install/quickstart)
> - [https://docs.rke2.io/install/ha](https://docs.rke2.io/install/ha)
> - [https://docs.rke2.io/add-ons/gpu_operators](https://docs.rke2.io/add-ons/gpu_operators)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. Worker 角色與本案配置

| 節點 | 平台 | 角色 | 標籤建議 |
|------|------|------|----------|
| `rke2-wkr-01` | VMware VM | 一般工作負載 / Run:ai system pods | `node-role.kubernetes.io/worker=true`, `workload=cpu` |
| `rke2-wkr-02` | VMware VM | 一般工作負載 / Run:ai system pods | 同上 |
| `rke2-gpu-01` | HGX XE9780（Bare-metal） | GPU 工作負載 | `workload=gpu`, `nvidia.com/gpu.present=true`（GPU Operator 自動加） |

> **設計理由**：CPU Worker 承載 Run:ai system pods（system node 需求：10 vCPU / 20 GB RAM / 50 GB），GPU Worker 專注於模型推論／訓練。詳見 [Run:ai Self-hosted v2.25 system requirements](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/install-using-helm/system-requirements.md)。

---

## 2. CPU Worker 加入（VM）

### 2.1 取得 token

於任一 Server 節點：

```bash
sudo cat /var/lib/rancher/rke2/server/node-token
# K10abc...::server:mySharedSecretToken
```

> 此檔內容**並非**單純 token，包含 cluster CA hash + token；agent 加入時可使用此完整字串，或直接使用 srv-01 設定中的 `token:` 值（兩者皆可）。

### 2.2 設定檔

於 worker 節點：

```bash
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml <<EOF
server: https://rke2.example.local:9345
token: mySharedSecretToken
node-label:
  - workload=cpu
EOF
```

### 2.3 安裝 agent

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE=agent INSTALL_RKE2_CHANNEL=stable sh -

sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
sudo journalctl -u rke2-agent -f
```

### 2.4 驗證

於 Server 節點：

```bash
kubectl get nodes
# rke2-wkr-01   Ready   <none>   3m   v1.35.5+rke2r1
# rke2-wkr-02   Ready   <none>   3m   v1.35.5+rke2r1

# 給 Worker 加上 role 標籤（純顯示用，不影響排程）
kubectl label node rke2-wkr-01 node-role.kubernetes.io/worker=true
kubectl label node rke2-wkr-02 node-role.kubernetes.io/worker=true
```

---

## 3. GPU Worker 加入（HGX Bare-metal）

### 3.1 前置確認（OS 層）

於 GPU 節點：

```bash
# 確認 nouveau 已停用
lsmod | grep nouveau   # 應無輸出

# 確認 GPU 硬體偵測
lspci | grep -i nvidia
```

> **是否預裝 NVIDIA Driver？**
> - **建議：不要預裝**，交由 GPU Operator 部署 driver container（簡化版本管理）。
> - **若已預裝**：須在 GPU Operator Helm values 設定 `driver.enabled: false`。

### 3.2 加入 RKE2 叢集（先當成普通 agent）

```bash
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml <<EOF
server: https://rke2.example.local:9345
token: mySharedSecretToken
node-label:
  - workload=gpu
node-taint:
  - nvidia.com/gpu=true:NoSchedule   # 避免非 GPU 工作負載排上 GPU 節點
EOF

curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE=agent INSTALL_RKE2_CHANNEL=stable sh -
sudo systemctl enable rke2-agent.service
sudo systemctl start rke2-agent.service
```

> **Taint 設計**：加上 `nvidia.com/gpu=true:NoSchedule` 強制只有明確 toleration 的 Pod 能排程到 GPU 節點，避免 Run:ai system pods 或一般工作負載佔用 GPU 資源。

### 3.3 部署 NVIDIA GPU Operator

來源：[https://docs.rke2.io/add-ons/gpu_operators](https://docs.rke2.io/add-ons/gpu_operators)

RKE2 採用內建的 helm-controller，可用 `HelmChart` CR 直接部署。

#### 3.3.1 選擇 GPU Operator 版本

| GPU Operator 版本 | RKE2 / containerd 要求 | 機制 | Run:ai 2.25 相容 |
|-------------------|------------------------|------|-------------------|
| v25.3.x | containerd 1.7+ | 修改 containerd config（volume-mounts strategy） | ⚠ **不在 Run:ai 2.25 Operator and Framework Versions 支援範圍**；舊版 Run:ai 請查對應版本 support matrix |
| v25.10.x | containerd 2.0+ | 修改 containerd config（CDI） | ✓ 支援範圍下限 |
| v26.3.x | containerd v1.7.30+、v2.1.x 或 v2.2.x | **NRI plugin**，**不需**修改 containerd config | ✓ 支援範圍上限（可選；部署前確認 release notes） |

**本案推薦**：RKE2 v1.35.5+rke2r1（內含 containerd v2.2.3）+ GPU Operator **v25.10.x**（落入 Run:ai 2.25 官方支援範圍 25.10–26.3）。

> **⚠ 版本決策依據**：Run:ai v2.25 官方支援的 GPU Operator 範圍為 **25.10–26.3**（[Run:ai Self-hosted v2.25 support matrix](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/support-matrix.md)，2026-05-26 查詢）。**以 support matrix「Operator and Framework Versions」表為最終依據**；若本文件版本與 support matrix 有衝突，以官方 support matrix 為準，並請向 NVIDIA/Run:ai support 確認。

> **v26.3.x 與 containerd 相容性**：v26.3 採用 NRI（Node Resource Interface）plugin 模式。依 [GPU Operator 26.3 Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.3/release-notes.html)，支援 containerd **v1.7.30+、v2.1.x 或 v2.2.x**。RKE2 v1.35.5+rke2r1 內含 containerd **v2.2.3**，在支援範圍內，NRI 模式技術上可行。**v26.3.x 為可選版本**；本案保守選擇仍建議 v25.10.x；若選用 v26.3.x，部署前請向 NVIDIA / Run:ai support 確認。

#### 3.3.2 透過 RKE2 helm-controller 部署

於 Server 節點：

```bash
sudo tee /var/lib/rancher/rke2/server/manifests/gpu-operator.yaml <<'EOF'
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: gpu-operator
  namespace: kube-system
spec:
  repo: https://helm.ngc.nvidia.com/nvidia
  chart: gpu-operator
  version: v25.10.1
  targetNamespace: gpu-operator
  createNamespace: true
  valuesContent: |-
    toolkit:
      env:
      - name: CONTAINERD_SOCKET
        value: /run/k3s/containerd/containerd.sock
EOF
```

> **`/var/lib/rancher/rke2/server/manifests/` 目錄**：RKE2 會自動套用此目錄中的 YAML（類似 `kubectl apply`）。詳見 [docs.rke2.io – Advanced](https://docs.rke2.io/advanced)。

#### 3.3.3 驗證 GPU 節點

```bash
# 等待 GPU Operator 部署完成（5–10 分鐘）
kubectl get pods -n gpu-operator -w

# 確認 nvidia 相關節點標籤（逐項列出）
kubectl get node rke2-gpu-01 --show-labels | tr ',' '\n' | grep -i nvidia
# 或：kubectl get node rke2-gpu-01 -o json | jq '.metadata.labels | with_entries(select(.key|test("nvidia")))'

# 確認 GPU 資源
kubectl get node rke2-gpu-01 -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'
# 應看到 8（HGX 8-GPU 機架；實際數量依硬體決定）
```

#### 3.3.4 nvidia runtime 驗證

```bash
# 容器運行時 binary 應存在（v25.10.x 與 v26.3.x 皆適用）
ls /usr/local/nvidia/toolkit/nvidia-container-runtime
```

**若使用 GPU Operator v25.10.x（修改 containerd config 模式）：**

```bash
# containerd config.toml 應已注入 nvidia runtime 設定
grep nvidia /var/lib/rancher/rke2/agent/etc/containerd/config.toml
```

> **注意**：GPU Operator 在套用 containerd 設定後會**重啟 containerd**，連帶觸發 RKE2 重啟。這是預期行為（[來源](https://docs.rke2.io/add-ons/gpu_operators)）。

**若使用 GPU Operator v26.3.x（NRI plugin 模式）：**

> NRI plugin **不修改** `containerd/config.toml`，也不建立 `nvidia` runtime class；上方 `grep` 指令無輸出屬正常，**不代表部署失敗**。請改以下列方式驗證：
>
> ```bash
> # 1. GPU Operator pods 全部 Running
> kubectl get pods -n gpu-operator
>
> # 2. GPU 資源已可分配
> kubectl get node rke2-gpu-01 -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'
>
> # 3. 執行測試 GPU Pod（見 §3.4）
> ```

### 3.4 測試 GPU Pod

```yaml
# gpu-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nbody-gpu-benchmark
  namespace: default
spec:
  restartPolicy: OnFailure
  tolerations:
    - key: nvidia.com/gpu
      operator: Equal
      value: "true"
      effect: NoSchedule
  containers:
    - name: cuda-container
      image: nvcr.io/nvidia/k8s/cuda-sample:nbody
      args: ["nbody", "-gpu", "-benchmark"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

```bash
kubectl apply -f gpu-test.yaml
kubectl logs -f nbody-gpu-benchmark
# 應看到 nbody benchmark 在 GPU 上執行的數據
```

---

## 4. Worker 移除流程

### 4.1 軟性移除（保留節點資料）

```bash
# 1. Cordon + Drain（排空工作負載）
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. 在節點上停止服務
sudo systemctl stop rke2-agent.service
sudo systemctl disable rke2-agent.service

# 3. 從叢集中刪除節點記錄
kubectl delete node <node-name>
```

### 4.2 徹底解除安裝

執行 RKE2 安裝時自動放置的解除腳本：

```bash
# tarball 安裝（Ubuntu 24.04 預設方式）
sudo /usr/local/bin/rke2-uninstall.sh

# RPM 安裝（RHEL/SLES）
sudo yum remove -y rke2-agent
```

> **不可逆操作**：`rke2-uninstall.sh` 會刪除 `/var/lib/rancher/rke2/`、`/etc/rancher/rke2/`、所有 systemd unit 與 binary。執行前確認意圖。

---

## 5. 加入失敗排查速查

| 症狀 | 對策 |
|------|------|
| Agent 卡在 `Cluster CA hash verification failed` | token 不匹配或叢集已重建；重新從 server `/var/lib/rancher/rke2/server/node-token` 取得完整 token |
| Agent 卡在 `dial tcp ...9345: i/o timeout` | LB 未轉發 9345；防火牆封鎖；server URL 拼錯 |
| Agent 加入後 node 顯示 `NotReady` | CNI 未就緒；檢查 `kubectl get pods -n kube-system | grep -E 'canal|cilium'` |
| GPU 節點無 `nvidia.com/gpu` 資源 | GPU Operator 未完成部署或 driver 安裝失敗；`kubectl logs -n gpu-operator -l app=nvidia-driver-daemonset` |
| GPU Pod 出現 `nvidia-smi: not found` | nvidia container runtime 未正確注入 containerd config；確認 `grep nvidia /var/lib/rancher/rke2/agent/etc/containerd/config.toml` |

---

## 6. 加入完成檢查清單

- [ ] 所有 Worker 在 `kubectl get nodes` 中顯示 Ready
- [ ] GPU Worker `nvidia.com/gpu` 資源大於 0
- [ ] CPU Worker 有 `workload=cpu` 標籤
- [ ] GPU Worker 有 `workload=gpu` 標籤與 `nvidia.com/gpu=true:NoSchedule` taint
- [ ] 測試 GPU Pod 可成功執行 `nbody -gpu`
- [ ] 從 CPU Worker 部署的測試 Pod **無法**被排程到 GPU Worker（驗證 taint 生效）
