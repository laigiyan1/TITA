# NVIDIA Run:ai 架構驗收項目

## 環境架構假設

> **⚠️ 硬體型號待確認**：驗收前請以採購 BOM / Dell Service Tag / Quote 確認實際 GPU 型號。
> 本案 GPU 節點為 **Dell PowerEdge XE9780**（注意：非 XE9680）。XE9780 可能搭載 H200 SXM5 或 B200（HGX 模組），
> 兩者的驅動版本、NVLink 頻寬、MIG Profile、韌體基準均不同，確認型號後須對應調整第 1.1、3.2、6 節數值。
>
> **NVIDIA ERA 參考架構對齊說明**：本案 XE9780 設計上對應 NVIDIA ERA 2-8-9-400 HGX 節點模式。
> 惟 Dell 官方 2-8-9-400 endorsed brief 中列出的參考平台為 **XE9680**，非 XE9780。
> XE9780 的 Dell AI Factory endorsement 狀態需由 Dell / NVIDIA SE 或最新官方文件另行確認。

| 節點 | 型號 | 角色 |
|------|------|------|
| GPU 運算節點 | **Dell PowerEdge XE9780**（GPU 型號待 BOM 確認） | Run:ai GPU Worker |
| 管理節點 | Dell PowerEdge R670 × 2 | K8s Control Plane / Run:ai Manager |

> **⚠️ HA 拓撲聲明**：2 台 R670 不構成完整 Kubernetes HA（etcd 需奇數成員以維持 quorum）。
> 本架構須選擇其一：(A) 補充第三台 Control Plane 節點、(B) 外部獨立 etcd 三節點叢集、或 (C) 明確聲明此為非完整 HA 架構並載入風險確認書。
> 第七節 HA 驗收項目以實際採用拓撲為準。

---

## 版本鎖定（驗收前必填）

> 版本未鎖定前不得進行第二節以後的驗收。廠商與客戶須共同確認下表並簽字。

| 元件 | 採購/部署版本 | 備註 |
|------|-------------|------|
| Run:ai 版本 | ＿＿＿＿＿＿ | 如 v2.24 |
| Kubernetes 版本 | ＿＿＿＿＿＿ | 需在 Run:ai Support Matrix 內 |
| NVIDIA GPU Operator 版本 | ＿＿＿＿＿＿ | 需與 K8s 版本相容 |
| Container Runtime | ＿＿＿＿＿＿ | containerd 或 CRI-O |
| CUDA Driver 版本 | ＿＿＿＿＿＿ | 依 GPU 型號對應 |
| 實際 GPU 型號 / SKU | ＿＿＿＿＿＿ | 由 BOM / Service Tag 確認 |

---

## 一、基礎設施層驗收

### 1.1 硬體健康確認
- [ ] GPU 運算節點所有 GPU 偵測正常（`nvidia-smi` 顯示 GPU 數量與型號符合採購 BOM，無 Xid Error）
- [ ] NVLink / NVSwitch 拓撲正確（`nvidia-smi topo -m` 確認全互聯，頻寬符合採購 GPU 規格）
- [ ] R670 × 2 記憶體、磁碟、網卡狀態無告警（iDRAC 無 Critical Event）
- [ ] BMC/iDRAC 遠端管理可存取

### 1.2 網路驗收
- [ ] 管理網路（OOB）：所有節點 iDRAC 互通
- [ ] 叢集內部網路（Data Plane）：R670 ↔ XE9780 MTU 9000 Jumbo Frame 設定正確
- [ ] InfiniBand / RoCEv2（**如有配備獨立 IB 介面**）：`ibstat` 顯示 Active，`ib_write_bw` 達標（≥ 200 Gb/s）；**單節點場景**：XE9780 節點內 GPU 通訊由 NVSwitch 完成，無需外部 IB 交換器，此項可標記 N/A 並附說明
- [ ] Run:ai Manager FQDN 解析正常（DNS A Record 可從叢集內外解析）；**注意：若 Control Plane 與第一個 Cluster 安裝於同一 Kubernetes 叢集（same-cluster 模式），Cluster 端不一定需要額外 FQDN，以實際部署模式為準**
- [ ] TLS 憑證有效（Local CA / Self-signed 已完成 trust chain 驗證，CA 已被叢集節點與用戶端瀏覽器/CLI 信任），`kubectl get ingress -n runai-backend` 顯示正確 Host
- [ ] 對外存取：節點可 Pull Container Registry（或確認 Air-gap 鏡像倉庫）

### 1.3 儲存驗收
- [ ] 共享儲存掛載正常，XE9780 讀寫速度符合規格
  - Dell PowerScale（NFS）：`mount -t nfs <powerscale-ip>:/ifs/ai /mnt/test && dd if=/dev/zero of=/mnt/test/test bs=1M count=1024`
  - NFS on R670（PoC）：`showmount -e <r670-ip>` 確認 Export 正常
- [ ] PersistentVolume StorageClass 建立完成並可動態佈建（`kubectl get sc` 確認有 Default StorageClass）

---

## 二、Kubernetes 叢集層驗收

### 2.1 版本與拓撲
- [ ] K8s 版本與「版本鎖定表」一致（`kubectl version -o json` 或 `kubectl get nodes -o wide` 確認 kubelet 版本），且在 Run:ai Support Matrix 支援範圍內
- [ ] 所有節點狀態為 `Ready`（`kubectl get nodes -o wide`）
- [ ] Control Plane 拓撲符合採購決定（3 Master / 外部 etcd 三節點 / 或已簽署非完整 HA 確認書）
- [ ] etcd 成員健康（`etcdctl member list`，所有成員 `started`，quorum 正常）

### 2.2 核心元件
- [ ] CoreDNS Pod 全部 Running；`nslookup kubernetes.default` 可解析
- [ ] CNI（Calico / Flannel）Pod 全部 Running，跨節點 Pod 網路互通
- [ ] Ingress Controller（如 nginx / traefik）部署完成，Ingress 規則可 Route 至 Run:ai UI；**same-cluster 模式下 Cluster 端可不需獨立 Ingress Controller，以 Run:ai Cluster System Requirements 及實際部署模式為準**

### 2.3 GPU 先決條件（Run:ai Prerequisites）
- [ ] NVIDIA GPU Operator 版本符合鎖定表，DaemonSet 全部 Running（`kubectl get pods -n gpu-operator`）
- [ ] Node Feature Discovery（NFD）偵測 GPU 型號標籤正確（`kubectl describe node <gpu-node> | grep nvidia`）
- [ ] DCGM Exporter Pod Running，`/metrics` 端點可回應 GPU 指標
- [ ] NVIDIA Device Plugin DaemonSet 狀態 Running，`nvidia.com/gpu` 資源顯示正確數量
- [ ] Container Runtime 符合鎖定表（`crictl version`）；GPU Operator / Toolkit 的 CDI 或 NRI 設定符合鎖定版本，GPU workload 可成功取得 GPU（新版 GPU Operator 預設啟用 CDI，啟用時 nvidia runtime class 不再設為預設 handler，以實際 GPU Operator 版本設定為準，不強制要求 default runtime 為 nvidia）
- [ ] Prometheus Operator 或 kube-prometheus-stack 部署完成，ServiceMonitor CRD 可用

---

## 三、Run:ai 平台層驗收

### 3.1 安裝完整性
- [ ] Run:ai Control Plane Helm Release 正常：`helm list -n runai-backend` 顯示 `deployed`
- [ ] runai-backend Namespace 所有 Pod Running：`kubectl get pods -n runai-backend`
- [ ] PVC 均已 Bound：`kubectl get pvc -n runai-backend`
- [ ] Ingress 建立完成且 TLS 正常：`kubectl get ingress -n runai-backend`
- [ ] Run:ai Cluster Agent Helm Release 正常：`helm list -n runai`，所有 Pod Running
- [ ] Run:ai UI 可正常登入，顯示叢集及 GPU 資源總數

### 3.2 CLI 驗收
- [ ] `runai version` 顯示版本符合鎖定表
- [ ] `runai whoami` 顯示登入帳號正確
- [ ] `runai cluster list` 顯示叢集狀態為 Connected
- [ ] `runai cluster set <cluster-name>` 可切換叢集

### 3.3 GPU 資源呈現
- [ ] UI 顯示 GPU 數量與型號符合採購 BOM，GPU 記憶體容量正確
- [ ] GPU 狀態頁無 Not-Ready 或 Unknown GPU

### 3.4 排程功能
- [ ] 提交 Interactive Job（Jupyter）：成功分配 GPU，Pod 進入 Running
- [ ] 提交 Training Job（單節點）：完成後狀態為 Succeeded
- [ ] Job 排隊（Queue）機制：當 GPU 滿載時，後提交 Job 進入 Pending，資源釋放後自動調度
- [ ] Job 優先權（Priority）：高優先 Job 可搶佔低優先 Job（Preemption 測試）

### 3.5 GPU 分割驗收（依 Node Pool 分開驗收，MIG 與 Fractions 不可在同一節點共存）

**MIG Node Pool**
- [ ] `nvidia-smi mig -lgip` 顯示已設定的 MIG Profile
- [ ] Run:ai UI 能識別各 MIG Slice 並顯示對應記憶體容量
- [ ] 提交指定 MIG Slice 大小的 Job，成功分配至正確 Slice

**GPU Fractions Node Pool**（與 MIG 節點分開）
- [ ] 提交 0.5 GPU Fraction Job，確認記憶體隔離正常（不同 Job 不互相 OOM）
- [ ] 同一張 GPU 上兩個 Fraction Job 各自取得設定上限記憶體

**Time-Slicing Policy**（搭配 GPU Fractions 使用）
- [ ] Time-Slicing 策略套用於指定 Node Pool
- [ ] 確認部署模式並告知使用者：
  - **Run:ai Time-Slicing**：建立於 GPU Fractions 之上，記憶體切分與隔離由 Fractions 提供，compute runtime 分享由 Time-Slicing 提供，兩者須搭配使用
  - **NVIDIA Device Plugin 原生 Time-Slicing**（若採用）：為純 compute 分時，**不提供記憶體隔離**，OOM 需由應用層控制
  - 驗收時明確記錄本環境採用哪種 Time-Slicing 模式

---

## 四、多租戶與資源管控驗收

- [ ] 建立至少 2 個 Project（部門/團隊），各自設定 GPU 配額
- [ ] Project A 超過配額時，Job 進入 Pending（不佔用 Project B 配額）
- [ ] Over-Quota（借用閒置資源）功能：A 閒置時 B 可超用，A 提交後 B Job 被搶佔
- [ ] Department 層級資源上限生效驗證
- [ ] Node Pool 分配：指定 Job 只能調度至特定節點

---

## 五、使用者與權限驗收

- [ ] SSO / LDAP / SAML 整合（若需要）：AD 帳號可登入 Run:ai UI
- [ ] RBAC 角色驗證：Admin / Researcher / Viewer 權限隔離正確
- [ ] Researcher 只能看到自己 Project 的 Job，無法跨 Project 查看

---

## 六、效能基準驗收

> GPU 型號確認後，依實際規格調整以下基準數值。

- [ ] **GPU 運算基準**：以 DCGM `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` 指標或框架 benchmark（FP16 GEMM / MLPerf Inference）確認單卡 Tensor Core 利用率 ≥ 90%
- [ ] **多卡通訊**：NCCL AllReduce Test（`nccl-tests`）頻寬達採購 GPU 型號 NVSwitch 規格值（H100 SXM 參考值 ≥ 900 GB/s；B200/B300 依規格書）
- [ ] **Job 啟動延遲**：從提交 Job 到 Pod Running ≤ 60 秒（不含 Image Pull）
- [ ] **排程吞吐量**：同時提交 20 個 Job，排程器在 5 分鐘內完成全部分配

---

## 七、高可用性與韌性驗收

> 本節驗收項目以第二節確認的 Control Plane 拓撲為前提，非完整 HA 架構者跳過第一項。

- [ ] **Control Plane HA**（完整 HA 架構適用）：關閉一台 Control Plane 節點，剩餘節點自動接管，現有 Job 不中斷
- [ ] **Run:ai Manager 重啟**：重啟 runai-backend namespace Pod，排程恢復正常，Job 狀態不丟失
- [ ] **GPU 節點重啟**：XE9780 重開機後，Node 自動重新加入叢集，Run:ai GPU Agent 自動恢復
- [ ] **Network Partition**：模擬暫時網路中斷後，叢集狀態一致性恢復

---

## 八、監控與告警驗收

- [ ] DCGM Exporter 指標正常收集，Prometheus 可查詢到 `DCGM_FI_DEV_GPU_UTIL`、`DCGM_FI_DEV_FB_USED`
- [ ] Run:ai 平台指標正常收集；依官方 Metrics API / Telemetry 輸出確認以下指標可查詢（Prometheus raw metric 名稱需現場查詢確認）：
  - Metrics API：`ALLOCATED_GPU`、`GPU_UTILIZATION`、`AVG_WORKLOAD_WAIT_TIME`
  - Telemetry：`ALLOCATED_GPUS`
  - 以實際部署版本 Run:ai Metrics and Telemetry 文件列出的名稱為準
- [ ] Grafana Dashboard 顯示 GPU 利用率、Job 數量、等待佇列長度
- [ ] 告警規則：GPU 利用率 < 20% 持續 15 分鐘觸發告警
- [ ] 告警規則：Job 等待超過 30 分鐘觸發告警
- [ ] 告警通知管道（Email / Slack / Teams）測試送達

---

## 九、日誌與稽核驗收

- [ ] Job 執行日誌可依 workload type 透過對應指令查閱（如 `runai workspace logs <name>`、`runai inference logs <name>`、`runai training pytorch logs <name>`），或透過 Run:ai UI 查閱
- [ ] 稽核日誌（Audit Log）記錄使用者操作（提交/刪除 Job、修改配額）
- [ ] 日誌保留期限符合需求（建議 ≥ 90 天）

---

## 十、文件與操作移交驗收

- [ ] 架構圖（含網路拓撲、VLAN 規劃、etcd 拓撲）交付
- [ ] 版本鎖定表已填寫完整並簽字確認
- [ ] Run:ai 管理操作手冊（建立 Project、提交 Job、調整配額）
- [ ] 監控儀表板操作說明
- [ ] 常見故障排除 SOP（GPU Not-Ready、排程卡住、節點離線）
- [ ] 管理員帳號移交並確認可正常登入

---

## 驗收通過門檻建議

| 分類 | 標準 |
|------|------|
| 第一類（必過） | 版本鎖定表填寫 + 一、二、三、四、五 全數通過，零 Critical 缺失 |
| 第二類（條件通過） | 六、七 允許 1 項次要缺失，附改善計畫 |
| 第三類（後續追蹤） | 八、九、十 可列 Punch List，限期 2 週內補交 |
