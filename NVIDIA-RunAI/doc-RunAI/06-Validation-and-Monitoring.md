# 驗證與監控

> 來源：https://run-ai-docs.nvidia.com/self-hosted/infrastructure-setup/procedures/system-monitoring  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/helm-install  
> 整理日期：2026-05-20  
> 目標版本：NVIDIA Run:ai v2.25

---

## 1. 安裝後驗證清單

### 1.1 Kubernetes 叢集健康

```bash
# 所有節點就緒
kubectl get nodes

# 系統 Pod 狀態（所有應為 Running 或 Completed）
kubectl get pods -A | grep -v Running | grep -v Completed
```

### 1.2 Control Plane 元件驗證

```bash
# Control Plane Pod 狀態
kubectl get pods -n runai-backend

# Deployment 狀態（DESIRED == AVAILABLE）
kubectl get deployment -n runai-backend

# StatefulSet（PostgreSQL 等資料庫元件）
kubectl get statefulset -n runai-backend

# Service 狀態
kubectl get svc -n runai-backend
```

### 1.3 Cluster 元件驗證

```bash
# Cluster Pod 狀態
kubectl get pods -n runai

# Cluster Sync Pod（確認 Control Plane 連線）
kubectl get pod -n runai | grep cluster-sync

# Run:ai Agent Pod
kubectl get pod -n runai | grep agent

# Deployment 狀態
kubectl get deployment -n runai
```

### 1.4 Prometheus 元件驗證

```bash
# Prometheus Operator 狀態
kubectl get deployment kube-prometheus-stack-operator -n monitoring

# Prometheus 實例
kubectl get prometheus -n runai

# AlertManager 實例
kubectl get alertmanager -n runai

# AlertManager 服務
kubectl get svc alertmanager-operated -n runai
```

### 1.5 GPU Operator 驗證

```bash
# GPU Operator Pod 狀態
kubectl get pods -n gpu-operator

# GPU 資源可見性
kubectl describe node <xe9780-node-name> | grep -A 5 "Allocatable:"
# 應顯示：nvidia.com/gpu: <數量>
```

---

## 2. 功能性驗證測試

### 2.1 GPU 工作負載測試（kubectl 方式）

```yaml
# gpu-test.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: runai-gpu-test
  namespace: runai
spec:
  template:
    spec:
      schedulerName: runai-scheduler    # 使用 Run:ai 排程器
      restartPolicy: Never
      containers:
      - name: cuda-test
        image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04
        resources:
          limits:
            nvidia.com/gpu: 1
```

```bash
kubectl apply -f gpu-test.yaml
kubectl wait -n runai --for=condition=complete job/runai-gpu-test --timeout=120s
kubectl logs -n runai job/runai-gpu-test
kubectl delete -n runai job/runai-gpu-test
```

期望輸出包含：`Test PASSED`

### 2.2 Run:ai CLI 驗證

```bash
# 登入並確認連線
runai login --url https://runai.<DOMAIN>
runai cluster list

# 提交測試 Job
runai config project <project-name>
runai submit cli-test \
  --image nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04 \
  --gpu 1

# 確認狀態
runai list jobs

# 清理
runai delete cli-test
```

---

## 3. 系統監控

### 3.1 Run:ai 內建告警（自動生效）

Run:ai 安裝後會自動設定 Prometheus 告警規則：

| 告警項目 | 條件 | 嚴重性 |
|----------|------|--------|
| Agent cluster info push 失敗 | Push rate 異常 | Critical |
| Agent cluster info pull 失敗 | Pull rate 異常 | Critical |
| Container 記憶體使用率 > 80% | 持續 10 分鐘 | Warning |
| Container 記憶體使用率 > 90% | 持續 10 分鐘 | Critical |
| Container 重啟次數 > 2 | 10 分鐘內 | Warning |
| CPU 使用率 > 80% | 持續 10 分鐘 | Warning |
| DaemonSet 部署失敗 | 任何時間 | Critical |
| 節點記憶體異常 | 異常時 | Critical |

### 3.2 監控指令

```bash
# 節點資源使用
kubectl top node

# Pod 資源使用
kubectl top pod -n runai
kubectl top pod -n runai-backend

# 特定 Deployment 詳細資訊
kubectl describe deployment <NAME> -n runai
kubectl describe deployment <NAME> -n runai-backend

# Pod Logs（疑難排解）
kubectl logs deployment/<NAME> -n runai
kubectl logs deployment/<NAME> -n runai-backend

# DaemonSet 狀態
kubectl describe daemonset <NAME> -n runai-backend
```

### 3.3 Run:ai 內建監控儀表板

> ⚠️ **v2.25 重要變更**：Run:ai v2.25 已**移除 legacy Grafana dashboards**（包含 Overview、Analytics、Consumption、Multi-cluster 四個舊儀表板）。  
> 來源：[What's New v2.25 – Analytics](https://run-ai-docs.nvidia.com/self-hosted/getting-started/whats-new/whats-new-2-25)

Run:ai v2.25 改用內建的 Overview / Analytics 類監控頁面（透過 Control Plane UI 存取）：

- GPU 使用率趨勢（Analytics 頁面）
- 工作負載等待時間與狀態（Overview 頁面）
- 節點資源分配狀況（Nodes 頁面）
- 專案配額使用情況（Projects 頁面）

> v2.25 Analytics dashboard 可見性已改為以 READ 權限控制；Department / Project 層級 filtering 亦於 Overview 頁面提供。

---

## 4. 疑難排解

### 4.1 Log 收集

```bash
# 收集所有 runai 命名空間的 Pod Log
for pod in $(kubectl get pods -n runai -o name); do
  echo "=== $pod ===" 
  kubectl logs $pod -n runai --tail=50
done

# 收集 runai-backend 命名空間的 Log
for pod in $(kubectl get pods -n runai-backend -o name); do
  echo "=== $pod ===" 
  kubectl logs $pod -n runai-backend --tail=50
done
```

### 4.2 常見問題診斷

| 症狀 | 診斷指令 | 說明 |
|------|----------|------|
| Cluster 顯示「Disconnected」 | `kubectl get pod -n runai \| grep cluster-sync` | 確認 cluster-sync Pod 狀態與 Log |
| GPU 工作負載 Pending | `kubectl describe pod <pod-name> -n runai` | 查看 Events，確認 GPU 資源是否可分配 |
| UI 無法存取 | `kubectl get ingress -n runai-backend` | 確認 Ingress 設定與 TLS 憑證 |
| 告警未觸發 | `kubectl get prometheus -n runai` | 確認 Prometheus 實例正確建立 |
| 節點記憶體不足 | `kubectl top node` | 確認節點實際記憶體使用量 |

### 4.3 預安裝診斷工具（安裝前執行）

Run:ai 提供 Pre-install Diagnostics 工具，可在安裝前測試環境相容性：

```bash
# 下載並執行診斷工具（連線環境）
# 詳細步驟請參閱官方文件
# https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/helm-install
```

> **待驗證**：預安裝診斷工具的確切下載 URL 請從 Run:ai UI 取得

---

## 5. 升級注意事項

> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/upgrade

- 升級前備份 PostgreSQL 資料庫
- 升級順序：Control Plane 先升，Cluster 後升
- 跨版本升級需逐版進行（如 2.23 → 2.24 → 2.25）
- 升級前確認新版本的 K8s 版本支援範圍未排除目前 K8s 版本

---

## 6. 參考資源

| 資源 | URL |
|------|-----|
| 系統監控官方文件 | https://run-ai-docs.nvidia.com/self-hosted/infrastructure-setup/procedures/system-monitoring |
| Log 收集指引 | https://run-ai-docs.nvidia.com/self-hosted/infrastructure-setup/procedures/logs-collection |
| 節點維護 | https://run-ai-docs.nvidia.com/self-hosted/infrastructure-setup/procedures/nodes-maintenance |
| 升級指引 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/installation/install-using-helm/upgrade |
