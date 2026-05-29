# NVIDIA GPU Operator 安裝指引

> 來源：https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/system-requirements  
> 整理日期：2026-05-20  
> 目標版本：GPU Operator v26.3.x（Run:ai v2.25 支援範圍：25.10 – 26.3）  
> **版本格式說明**：Run:ai Support Matrix 使用 YY.MM 格式（如 25.10 = 2025年10月）；GPU Operator 安裝指令使用含小版本號（如 v26.3.1）。v26.3.x 屬於 26.3 系列，在 Run:ai v2.25 支援範圍內。  
> **部署前請至 [GPU Operator Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html) 確認最新版本在 Run:ai Support Matrix 支援範圍內**

---

## 1. 概述

NVIDIA GPU Operator 是 Run:ai 的**必要先決條件**，負責：
- 自動化 GPU 節點的 NVIDIA Driver 安裝與管理
- 部署 NVIDIA Container Toolkit（讓容器能使用 GPU）
- 部署 DCGM Exporter（GPU 指標收集）
- 節點特徵發現（Node Feature Discovery）

**安裝順序**：GPU Operator 必須在 Run:ai 安裝之前完成。

---

## 2. 先決條件

### 2.1 作業系統要求

- 所有 GPU Worker 節點必須執行**相同 OS 版本**
- Ubuntu 22.04 LTS（本案使用）

> **重要**：Ubuntu 22.04 上建議停用自動核心更新，以防升級至不相容的核心版本：
> ```bash
> # 停用 unattended-upgrades 自動核心更新
> sudo apt-mark hold linux-image-generic linux-headers-generic
> ```

### 2.2 Container Runtime

GPU Worker 節點需配置 Containerd 或 CRI-O，且需正確設定 Cgroup Driver：

```bash
# 確認 containerd 已安裝並使用 SystemdCgroup
containerd config default | grep SystemdCgroup
# 應輸出：SystemdCgroup = true
# 若為 false，需修改 /etc/containerd/config.toml
```

### 2.3 Helm 版本

需 Helm 3.x（建議 3.14+）：

```bash
helm version
```

---

## 3. 安裝步驟

### 步驟 1：新增 NVIDIA Helm Repository

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```

### 步驟 2：建立命名空間並設定 Pod Security

```bash
kubectl create ns gpu-operator
kubectl label --overwrite ns gpu-operator \
  pod-security.kubernetes.io/enforce=privileged
```

### 步驟 3：確認 Node Feature Discovery 狀態

```bash
# 確認 NFD 是否已存在（若已安裝，需告知 GPU Operator 跳過）
kubectl get nodes -o json | \
  jq '.items[].metadata.labels | keys | any(startswith("feature.node.kubernetes.io"))'
# 若回傳 true：NFD 已安裝
# 若回傳 false：NFD 未安裝，GPU Operator 會自動安裝
```

### 步驟 4：安裝 NVIDIA GPU Operator

> **版本選擇**：Run:ai v2.25 Support Matrix 列出支援範圍 25.10–26.3。  
> ⚠️ **生命週期警告**：
> - **25.10.x**：在 [NVIDIA GPU Operator 生命週期政策](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html) 中已列為 **Deprecated**（僅提供 critical bug 與 CVE 修補，不建議新部署選用）  
> - **26.3.x**：目前 Current Supported 版本，**建議選用**  
> - 若現場選用 25.10.x，需注意其 Deprecated 狀態，並評估升級計畫  
> 部署前至 [GPU Operator Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html) 與 [Run:ai Support Matrix](https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/support-matrix) 交叉確認。  
> 截至 2026-05 已知可用版本：v26.3.1（屬 26.3 系列，在支援範圍內且為 Current）。  
> 來源：[GPU Operator Platform Support / Lifecycle](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html)

#### 標準安裝（裸機 Ubuntu 22.04，Driver 由 Operator 管理）

```bash
GPU_OPERATOR_VERSION="v26.3.1"   # 部署前確認為最新相容版本

helm install --wait --generate-name \
    -n gpu-operator --create-namespace \
    nvidia/gpu-operator \
    --version=${GPU_OPERATOR_VERSION}
```

#### 生產環境建議安裝（含 CDI 與 DCGM 匯出器）

```bash
helm install --wait --generate-name \
    -n gpu-operator --create-namespace \
    nvidia/gpu-operator \
    --version=${GPU_OPERATOR_VERSION} \
    --set driver.kernelModuleType=auto \
    --set cdi.enabled=true \
    --set dcgmExporter.enabled=true
```

#### 若 GPU 節點已預裝 NVIDIA Driver

```bash
# 跳過 Driver 安裝（使用已存在的 Driver）
helm install --wait --generate-name \
    -n gpu-operator --create-namespace \
    nvidia/gpu-operator \
    --version=${GPU_OPERATOR_VERSION} \
    --set driver.enabled=false
```

> **待驗證**：Dell XE9780 是否已預裝 NVIDIA Driver，需現場確認後決定使用哪個安裝方式

---

## 4. 安裝驗證

### 步驟 1：確認所有 GPU Operator Pod 正常

```bash
kubectl get pods -n gpu-operator
```

**期望輸出**（所有 Pod 應為 Running 或 Completed）：
```
NAME                                                          READY   STATUS
gpu-operator-xxxxx                                            1/1     Running
gpu-operator-node-feature-discovery-master-xxxxx             1/1     Running
gpu-operator-node-feature-discovery-worker-xxxxx             1/1     Running
nvidia-container-toolkit-daemonset-xxxxx                      1/1     Running
nvidia-dcgm-exporter-xxxxx                                    1/1     Running
nvidia-device-plugin-daemonset-xxxxx                         1/1     Running
nvidia-driver-daemonset-xxxxx                                1/1     Running
```

### 步驟 2：確認 GPU 節點標籤

```bash
kubectl get node <xe9780-node-name> -o json | \
  jq '.metadata.labels | with_entries(select(.key | startswith("nvidia")))'
```

應看到類似 `nvidia.com/gpu.count`、`nvidia.com/gpu.product` 等標籤。

### 步驟 3：執行 CUDA 測試工作負載

建立測試檔案 `cuda-test.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vectoradd
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-vectoradd
    image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04"
    resources:
      limits:
        nvidia.com/gpu: 1
```

```bash
kubectl apply -f cuda-test.yaml
kubectl wait --for=condition=complete pod/cuda-vectoradd --timeout=120s
kubectl logs pod/cuda-vectoradd
kubectl delete -f cuda-test.yaml
```

**期望輸出**：
```
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

---

## 5. 常見問題

| 問題 | 可能原因 | 處理方式 |
|------|----------|----------|
| Driver Daemonset 持續 Pending | 核心版本不相容 | 確認 Ubuntu 核心版本符合 Driver 支援清單 |
| CUDA 測試失敗 | GPU 資源未正確識別 | 檢查 device-plugin daemonset 是否 Running |
| NFD 標籤未出現 | NFD 安裝失敗 | 查看 `kubectl logs -n gpu-operator` |
| Driver 啟動超時 | 硬體初始化慢 | 增加 `--set driver.startupProbe.failureThreshold=60` |

---

## 6. 下一步

GPU Operator 安裝並驗證完成後，繼續安裝 Run:ai：

→ [04-RunAI-Installation.md](04-RunAI-Installation.md)
