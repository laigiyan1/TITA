# 07 – RKE2 版本升級與回滾

> 來源：
> - [https://docs.rke2.io/upgrades/manual](https://docs.rke2.io/upgrades/manual)
> - [https://docs.rke2.io/upgrades/automated](https://docs.rke2.io/upgrades/automated)
> - [https://docs.rke2.io/upgrades/roll-back](https://docs.rke2.io/upgrades/roll-back)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. 升級基本原則

來源：[https://docs.rke2.io/upgrades/manual](https://docs.rke2.io/upgrades/manual)

1. **先升 Server，再升 Agent**：所有 server 升級完成後才開始升 agent。
2. **遵守 Kubernetes 版本偏差政策**：**不可跳過 minor 版本**（例如 1.33 → 1.35 中間必須經過 1.34）。
3. **同一 minor 內可直接升 patch**：例如 1.35.3 → 1.35.5 安全。
4. **升級前必做快照**：詳見 [06-Backup-and-Restore.md §3](06-RKE2-Backup-and-Restore.md)。

---

## 2. 升級前置作業

```bash
# 1. 手動快照
sudo /var/lib/rancher/rke2/bin/rke2 etcd-snapshot save \
  --name pre-upgrade-$(date +%Y%m%d-%H%M)

# 2. 備份 token
sudo cp /var/lib/rancher/rke2/server/token /backup/token-pre-upgrade.bak

# 3. 紀錄當前版本（rollback 用）
/var/lib/rancher/rke2/bin/rke2 --version > /backup/rke2-version-pre-upgrade.txt
kubectl get nodes -o wide > /backup/nodes-pre-upgrade.txt

# 4. 確認叢集健康
kubectl get nodes
kubectl get pods -A | grep -vE 'Running|Completed'   # 不應有輸出
```

---

## 3. 手動升級（推薦：小規模叢集）

### 3.1 升級 Server（一次一個）

於 srv-01 執行：

```bash
# 1. Drain（保留 daemonset；evict 一般 Pod）
kubectl drain rke2-srv-01 --ignore-daemonsets --delete-emptydir-data

# 2. 安裝新版本
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_CHANNEL=stable sh -
# 或指定版本
# curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_VERSION=v1.35.6+rke2r1 sh -

# 3. 重啟服務
sudo systemctl restart rke2-server

# 4. 等待節點 Ready
kubectl get node rke2-srv-01 -w
# 確認 VERSION 為新版本

# 5. Uncordon
kubectl uncordon rke2-srv-01
```

> **注意**：drain 前先確認 ingress/run:ai 等對 srv-01 的依賴狀態。Server 節點預設無工作負載，主要影響仍是控制平面短暫切換。

於 srv-02、srv-03 重複上述步驟，**每次一個**。

### 3.2 升級 Agent（CPU Worker）

```bash
# 1. Drain
kubectl drain rke2-wkr-01 --ignore-daemonsets --delete-emptydir-data

# 2. 安裝新版
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE=agent INSTALL_RKE2_CHANNEL=stable sh -

# 3. 重啟
sudo systemctl restart rke2-agent

# 4. 等待 Ready 後 Uncordon
kubectl get node rke2-wkr-01 -w
kubectl uncordon rke2-wkr-01
```

### 3.3 升級 GPU Worker（HGX）

GPU Worker 需要額外考量：

```bash
# 1. Drain（GPU Pod 必須完整 evict）
kubectl drain rke2-gpu-01 --ignore-daemonsets --delete-emptydir-data --force

# 2. 升級 RKE2 agent
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE=agent INSTALL_RKE2_CHANNEL=stable sh -
sudo systemctl restart rke2-agent

# 3. GPU Operator 會自動重建 nvidia 相關 pods；等待全部 Ready
kubectl get pods -n gpu-operator -w

# 4. 驗證 GPU 資源仍可用
kubectl get node rke2-gpu-01 -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'

# 5. Uncordon
kubectl uncordon rke2-gpu-01
```

> **GPU Operator 與 RKE2 升級相依性**：跨 minor 版本升級 RKE2（例如 1.35 → 1.36）可能伴隨 containerd 大版本變更，**應同步檢查 GPU Operator 是否需要對應升級**（例如 containerd 2.1 才支援 GPU Operator v26.3 的 NRI 模式）。

---

## 4. RPM 升級（如用 RPM 安裝）

來源：[https://docs.rke2.io/upgrades/manual](https://docs.rke2.io/upgrades/manual)

```bash
# Server
sudo yum update rke2-server -y
sudo systemctl restart rke2-server

# Agent
sudo yum update rke2-agent -y
sudo systemctl restart rke2-agent
```

> Ubuntu 24.04 一般採用 tarball 安裝（無官方 .deb），本案不適用 RPM。

---

## 5. 自動升級（system-upgrade-controller）

來源：[https://docs.rke2.io/upgrades/automated](https://docs.rke2.io/upgrades/automated)

### 5.1 安裝 controller

```bash
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/crd.yaml \
              -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
```

### 5.2 建立 Server Plan

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1                # **必為 1**：一次只升一個 server
  cordon: true
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/control-plane, operator: In, values: ["true"]}
  serviceAccountName: system-upgrade
  channel: https://update.rke2.io/v1-release/channels/stable
  upgrade:
    image: rancher/rke2-upgrade
```

### 5.3 建立 Agent Plan

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 1                # 依工作負載容忍度決定
  cordon: true
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  serviceAccountName: system-upgrade
  prepare:
    args: ["prepare", "server-plan"]   # 等 server plan 完成
    image: rancher/rke2-upgrade
  channel: https://update.rke2.io/v1-release/channels/stable
  upgrade:
    image: rancher/rke2-upgrade
```

> `prepare` 區塊確保 agent 在所有 server 升級完成後才開始。

### 5.4 鎖定升級時間視窗

可加入 `window` 限制執行時段（例如僅週日凌晨）。詳見官方文件。

---

## 6. 回滾流程

來源：[https://docs.rke2.io/upgrades/roll-back](https://docs.rke2.io/upgrades/roll-back)

> 回滾**必須有**：升級前同一 K8s minor 版本的 etcd 快照 + token。

### 6.1 步驟

**第 1 步：Drain 所有節點**

```bash
for n in rke2-srv-01 rke2-srv-02 rke2-srv-03 rke2-wkr-01 rke2-wkr-02 rke2-gpu-01; do
  kubectl drain $n --ignore-daemonsets --delete-emptydir-data --force
done
```

**第 2 步：在每個節點停止 RKE2**

```bash
# Server / Agent 皆使用此腳本
sudo /usr/local/bin/rke2-killall.sh
```

> **`rke2-killall.sh` 行為**：強制終止 rke2 主進程、containerd、所有 pod 與 mount，但**不**解除 systemd unit、不刪除 binary、不刪除 `/var/lib/rancher/rke2/` 資料。後續第 3 步的 install script 會替換 binary，systemd unit 仍可繼續使用。

**第 3 步：在每個節點降級 binary**

```bash
# Server
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_VERSION=v1.35.5+rke2r1 sh -
# Agent
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE=agent INSTALL_RKE2_VERSION=v1.35.5+rke2r1 sh -
```

**第 4 步：在 srv-01 還原 etcd 快照**

```bash
sudo /var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/<pre-upgrade-snapshot>
```

**第 5 步：依序重啟**

```bash
# srv-01
sudo systemctl start rke2-server

# srv-02、srv-03（先清空 etcd dir）
sudo rm -rf /var/lib/rancher/rke2/server/db
sudo systemctl start rke2-server

# 所有 agent
sudo systemctl start rke2-agent
```

**第 6 步：驗證**

```bash
/var/lib/rancher/rke2/bin/rke2 --version
kubectl get nodes
kubectl get pods -A
```

> **回滾會覆寫所有 etcd 資料**：升級後的任何變更（新部署、namespace、設定）皆會遺失。

---

## 7. 升級風險與緩解

| 風險 | 緩解 |
|------|------|
| API 短暫不可用 | drain 一個 server 期間，其他 2 個仍可服務 → HA 設計關鍵 |
| containerd 升級導致 Pod 重啟 | 預期行為；確保關鍵應用有多 replica |
| GPU Operator 與 containerd 版本不相容 | 升級前查 [Run:ai support matrix](https://run-ai-docs.nvidia.com/) + [GPU Operator release notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/) |
| Ingress 預設變更（v1.35 → v1.36：nginx → traefik） | **舊叢集升級保留原 ingress**；新叢集才會用新預設。升級前確認此行為 |
| etcd 版本不相容 | RKE2 同 minor 內 etcd 相容；跨 minor 由 RKE2 處理 |

---

## 8. 升級檢查清單

### 8.1 升級前

- [ ] 已完成手動快照與 token 備份
- [ ] 已記錄當前版本與節點狀態
- [ ] 已通知工作負載擁有者（特別是 Run:ai team）
- [ ] 已確認目標版本與 Run:ai / GPU Operator 相容
- [ ] 已選定升級時段（低峰期）

### 8.2 升級後

- [ ] 所有節點 `kubectl get nodes` 顯示新版本且 Ready
- [ ] 系統 Pod 全部 Running
- [ ] GPU 資源仍可用
- [ ] 抽樣應用功能驗證通過
- [ ] 自動快照排程仍生效
