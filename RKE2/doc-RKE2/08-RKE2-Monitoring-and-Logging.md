# 08 – 監控與日誌

> 來源：
> - [https://docs.rke2.io/advanced](https://docs.rke2.io/advanced)
> - [https://docs.rke2.io/install/requirements](https://docs.rke2.io/install/requirements)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. 日誌位置

來源：[https://docs.rke2.io/advanced](https://docs.rke2.io/advanced)

| 來源 | 位置 |
|------|------|
| RKE2 systemd 主進程 | `journalctl -u rke2-server` / `-u rke2-agent`，亦會落入 `/var/log/syslog` |
| 控制平面 static pod | `kubectl logs -n kube-system <pod-name>` 或 `crictl logs <container-id>` |
| containerd | `journalctl -u rke2-server`（內含 containerd 輸出） |
| 個別容器 | containerd 管理；可 `kubectl logs` 或 `crictl logs` |

### 1.1 常用 journalctl 指令

```bash
# 最近 1 小時錯誤
sudo journalctl -u rke2-server --since "1 hour ago" -p err

# 即時追蹤
sudo journalctl -u rke2-server -f

# 限定特定關鍵字
sudo journalctl -u rke2-server | grep -i "etcd"

# 過去 24h 完整匯出供分析
sudo journalctl -u rke2-server --since "24 hours ago" > rke2-server.log
```

### 1.2 crictl 操作（containerd 互動）

```bash
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

crictl ps                      # 執行中容器
crictl pods                    # Pod 列表
crictl logs <container-id>     # 容器日誌
crictl images                  # 映像
crictl stats                   # 資源使用率
```

> RKE2 已預設配置 `crictl.yaml`，連線 socket `/run/k3s/containerd/containerd.sock`。

---

## 2. 內建監控元件

### 2.1 metrics-server

RKE2 預設啟用 metrics-server（chart 名 `rke2-metrics-server`），提供 `kubectl top` 功能：

```bash
kubectl top nodes
kubectl top pods -A
```

### 2.2 kubelet metrics

每節點埠 `10250/tcp` 暴露 kubelet metrics（需 TLS + RBAC）。

### 2.3 etcd metrics

每 Server 埠 `2381/tcp` 暴露 etcd metrics（Prometheus 格式）。

---

## 3. Prometheus 整合（Run:ai 必要前置）

來源：[Run:ai Self-hosted v2.25 system requirements](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/install-using-helm/system-requirements.md)

> "NVIDIA Run:ai cluster requires Prometheus to be installed on the Kubernetes cluster."

**RKE2 預設不含 Prometheus**，須自行部署。建議：

### 3.1 kube-prometheus-stack（Prometheus Operator）

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 部署
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

> **與 RKE2 控制平面 metrics 整合**：因 RKE2 控制平面為 static pods（在主機網路命名空間內，無對應 Kubernetes Service），常見作法有兩種：
> 1. **PodMonitor**：對應到 static pod 的 label（例如 `component=kube-controller-manager`），抓取對應節點 IP 的 `10257`（controller-manager）/ `10259`（scheduler）/ `2381`（etcd metrics）。
> 2. **Headless Service + 手動 Endpoints**：建立 Service 與固定 Endpoints 指向 server 節點 IP，再以 ServiceMonitor 套用。
>
> 詳細實作請參考 [Rancher Monitoring 文件](https://ranchermanager.docs.rancher.com/) 與 kube-prometheus-stack chart 的 `kubeControllerManager.endpoints` / `kubeScheduler.endpoints` 設定範例。

### 3.2 GPU 監控（DCGM Exporter）

GPU Operator 預設部署 `nvidia-dcgm-exporter`，暴露 GPU 利用率、溫度、功耗等指標於每節點 `:9400/metrics`。建議配 ServiceMonitor 由 Prometheus 抓取。

---

## 4. 日誌彙整建議

RKE2 不內建集中式日誌；建議方案：

| 方案 | 適用情境 |
|------|----------|
| **Loki + Promtail / Grafana** | 與 Prometheus / Grafana 配套；查詢語法統一 |
| **Elastic Stack（Elasticsearch + Filebeat / Fluent Bit）** | 已有 ELK 基礎建設 |
| **vector + S3 / 物件儲存** | 高吞吐 / 成本敏感 |

Fluent Bit DaemonSet 為最常見選擇，從 `/var/log/containers/` 收集所有 Pod 日誌。

---

## 5. 健康檢查腳本範例

可設定 Prometheus alerting 或 cron 監控以下指標：

```bash
#!/bin/bash
# rke2-health-check.sh — 建議每 5 分鐘執行一次
set -e
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin

# 1. API 可用性
kubectl get --raw=/healthz | grep -q ok || { echo "API DOWN"; exit 1; }

# 2. Node Ready 數量
READY=$(kubectl get nodes --no-headers | awk '$2=="Ready"' | wc -l)
TOTAL=$(kubectl get nodes --no-headers | wc -l)
[ "$READY" -eq "$TOTAL" ] || echo "WARN: $READY/$TOTAL nodes ready"

# 3. etcd 健康
kubectl -n kube-system exec etcd-rke2-srv-01 -- etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health --cluster 2>&1 | grep -q "is healthy" || echo "WARN: etcd unhealthy"

# 4. 最新 snapshot 時間（檢查超過 24h）
LATEST=$(ls -t /var/lib/rancher/rke2/server/db/snapshots/ | head -1)
AGE=$(( $(date +%s) - $(stat -c %Y /var/lib/rancher/rke2/server/db/snapshots/$LATEST) ))
[ "$AGE" -lt 86400 ] || echo "WARN: latest snapshot is older than 24h"
```

---

## 6. 推薦告警項目

| 告警 | 條件 | 嚴重性 |
|------|------|--------|
| Node NotReady | 任一節點超過 5 min | P1 |
| API server 不可用 | `/healthz` 失敗連續 3 次 | P1 |
| etcd member 失聯 | endpoint health 失敗 | P1 |
| etcd DB size | > 80% 配額（預設 2GB） | P2 |
| Pod CrashLoopBackOff | kube-system / gpu-operator namespace | P2 |
| 自動快照失敗 | 最新 snapshot > 24h | P2 |
| GPU 溫度 | DCGM > 85°C 持續 5 min | P2 |
| GPU 不可用 | `nvidia.com/gpu` allocatable = 0 | P1 |

---

## 7. 日誌與監控檢查清單

- [ ] systemd 日誌啟用持久化（`/etc/systemd/journald.conf` 設 `Storage=persistent`）
- [ ] 設定 logrotate 或限制 journald 大小（避免磁碟塞滿）
- [ ] 已部署 Prometheus（Run:ai 前置）
- [ ] 已部署 DCGM Exporter ServiceMonitor
- [ ] 已建立基本告警規則
- [ ] 已建立 snapshot age 監控
- [ ] 已建立健康檢查腳本（cron 或 Prometheus blackbox）
