# NVIDIA Run:ai 教育訓練大綱

> 適用對象：完全未接觸過 NVIDIA / Run:ai 的人員（IT 維運、研究員、部門主管）
> 訓練時間：2 小時（120 分鐘）
> 訓練方式：簡報講解 + 系統 Demo + Q&A
> 客戶環境：Dell XE9780（GPU）× 1 + Dell R670 × 2，Self-hosted K8s 部署

---

## 訓練目標

完成本次訓練後，學員應能：

- [ ] 說明 GPU 在 AI 工作負載中的角色，以及為何需要 Run:ai
- [ ] 理解客戶環境中三台伺服器各自扮演的角色
- [ ] 從 Run:ai UI 查看 GPU 使用狀態與工作負載
- [ ] 透過 Run:ai 提交、監控、停止 GPU 工作負載
- [ ] 了解現有架構未來可如何擴展

---

## 時程總覽

| 時間 | 模組 | 內容摘要 |
|------|------|----------|
| 00:00 – 00:10 | **開場** | 訓練目標說明、學員背景確認 |
| 00:10 – 00:30 | **模組一** | 為什麼需要 GPU 排程？AI 基礎架構的挑戰 |
| 00:30 – 00:55 | **模組二** | NVIDIA Run:ai 架構與核心概念 |
| 00:55 – 01:10 | **模組三** | 客戶環境對應：三台伺服器的角色分工 |
| 01:10 – 01:15 | **休息** | |
| 01:15 – 01:45 | **模組四** | 常用操作 Demo（Run:ai UI + CLI） |
| 01:45 – 01:55 | **模組五** | 未來擴展性與演進路徑 |
| 01:55 – 02:00 | **Q&A** | 問題解答與總結 |

---

## 開場（00:00 – 00:10）｜10 分鐘

### 目的
建立共同語境，了解學員背景差異（IT 維運 vs 研究員 vs 管理者）

### 內容
- 自我介紹與訓練目標說明
- 學員背景確認（簡單舉手）：
  - 是否聽過 Kubernetes？
  - 是否執行過 AI / 深度學習訓練？
  - 是否管理過多人共用資源的環境？
- 說明本次訓練**不需要**程式基礎，重點在「看得懂、用得到」

---

## 模組一：為什麼需要 GPU 排程？（00:10 – 00:30）｜20 分鐘

### 核心訊息
> GPU 是昂貴且稀缺的資源，沒有管理工具就像沒有停車場系統的停車場。

### 1-1｜GPU 與 CPU 的差異（5 分鐘）

- CPU：少量核心、擅長複雜邏輯、處理一般程式
- GPU：數千個小核心、擅長大量平行運算、訓練 AI 模型的主力
- 類比：CPU 是主廚，GPU 是整個廚房的流水線員工
- Dell PowerEdge XE9780 是什麼？→ 搭載 NVIDIA HGX 模組（8× 資料中心 GPU + NVSwitch 全互聯）的高效能 AI 伺服器，設計上參考 NVIDIA ERA 2-8-9-400 HGX 節點模式（2 CPU / 8 GPU / 高速 NIC），定位於 Dell AI Factory 企業 AI 基礎架構生態系（⚠️ Dell 官方 2-8-9-400 brief 明確列出的參考平台為 XE9680；本案 XE9780 的 Dell AI Factory endorsement 狀態需以 Dell / NVIDIA SE 或最新官方文件確認）

### 1-2｜沒有 GPU 排程的世界（5 分鐘）

常見問題情境（帶入學員日常）：

| 問題 | 沒有 Run:ai 時 | 有 Run:ai 時 |
|------|----------------|--------------|
| 多人搶同一張 GPU | 先到先得、互相等待 | 自動排隊、公平分配 |
| GPU 閒置沒人知道 | 看不到誰佔用 | 即時儀表板顯示 |
| 工作執行中當掉 | 需人工重啟 | 自動重新排程 |
| 不同專案資源不均 | 憑感覺分配 | 設定配額強制執行 |

### 1-3｜NVIDIA Run:ai 解決什麼問題（10 分鐘）

- **定義**：AI Operations Platform，GPU 資源的「調度中心」
- **三大核心能力**：
  1. **看得到**：即時 GPU 使用率、誰在用、用多少
  2. **管得到**：設定哪個部門/專案能用多少 GPU
  3. **排得到**：工作負載自動排隊、閒置 GPU 自動分配

- 視覺化說明：
```
研究員 A 提交訓練任務（需要 4 GPU）
        ↓
   Run:ai 檢查配額
        ↓
  XE9780 有空閒 GPU？ → 是 → 立即啟動
                      → 否 → 排隊等待，有空就自動啟動
```

---

## 模組二：NVIDIA Run:ai 架構與核心概念（00:30 – 00:55）｜25 分鐘

### 2-1｜兩個主要元件（10 分鐘）

Run:ai 由兩個部分組成，均運行在 Kubernetes（容器管理平台）之上：

```
┌──────────────────────────────────────┐
│         Run:ai Control Plane         │  ← 管理中心
│  Web UI / API / 儀表板 / 帳號管理    │    （住在 R670）
│  PostgreSQL 資料庫 + Keycloak 認證   │
└──────────────┬───────────────────────┘
               │ 加密連線（只傳 Metadata，不傳資料）
               ▼
┌──────────────────────────────────────┐
│          Run:ai Cluster              │  ← 執行引擎
│  GPU 排程器 / 工作負載管理           │    （住在 XE9780）
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│     Dell XE9780（GPU Worker）        │  ← 真正跑 AI 的地方
│  NVIDIA GPU × N 張                  │
└──────────────────────────────────────┘
```

**重點說明**：
- Control Plane = 指揮部（管理、監控、接收指令）
- Cluster = 執行部隊（實際調度 GPU 資源）
- 安全設計：Cluster 只傳送狀態資訊到 Control Plane，**用戶資料/模型不離開本地**

### 2-2｜資源管理階層（10 分鐘）

Run:ai 使用三層結構管理資源，類比公司組織：

```
Cluster（整個公司的 GPU 池）
  └── Department（部門）— 例：研究部、產品部
        └── Project（專案）— 例：NLP 專案、CV 專案
              └── Workload（工作）— 例：模型訓練、推論服務
```

| 層級 | 類比 | 設定內容 |
|------|------|----------|
| Cluster | 公司 | 總 GPU 數量 |
| Department | 部門 | 部門最大 GPU 配額 |
| Project | 專案 | 專案 GPU 配額（可設小數，如 1.5 張） |
| Workload | 工作 | 單次任務需要幾張 GPU |

**GPU 分數（GPU Fraction）概念**：
- 一張 GPU 可以拆分給多個小工作負載使用
- 例：1 張 GPU → 0.5 給推論服務 A + 0.5 給推論服務 B
- 大幅提升 GPU 利用率

### 2-3｜工作負載類型（5 分鐘）

| 類型 | 用途 | 特性 |
|------|------|------|
| **Training** | 模型訓練 | 有明確結束時間，完成後釋放 GPU |
| **Workspace** | 互動式開發 | 長時間執行（Jupyter / VS Code） |
| **Inference** | 模型推論服務 | 持續運行，對外提供 API |

---

## 模組三：客戶環境對應（00:55 – 01:10）｜15 分鐘

### 3-1｜三台伺服器的角色分工（8 分鐘）

```
┌─────────────────────────────────────────────────────────────┐
│                     客戶資料中心                             │
│                                                             │
│  ┌─────────────────┐   ┌─────────────────┐                 │
│  │  Dell R670 #1   │   │  Dell R670 #2   │                 │
│  │  K8s Master     │   │  K8s Master     │                 │
│  │  Run:ai         │   │  (HA 備援)      │                 │
│  │  Control Plane  │   │                 │                 │
│  │  （管理 Web UI）│   │                 │                 │
│  └────────┬────────┘   └────────┬────────┘                 │
│           │                     │                           │
│           └──────────┬──────────┘                           │
│                      │ Kubernetes 叢集                       │
│                      ▼                                       │
│           ┌──────────────────────┐                          │
│           │   Dell XE9780        │                          │
│           │   K8s Worker Node    │                          │
│           │   Run:ai Cluster     │                          │
│           │   NVIDIA GPU（HGX）  │ ← AI 工作實際在這裡執行  │
│           └──────────────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

| 伺服器 | 規格重點 | 在 Run:ai 的角色 | 日常狀況 |
|--------|----------|-----------------|----------|
| R670 #1 | 無 GPU，CPU/RAM 密集 | Control Plane 主節點 | 提供 Web UI、處理 API 請求 |
| R670 #2 | 無 GPU | HA 備援節點 | R670 #1 故障時自動接手 |
| XE9780 | NVIDIA HGX GPU × N | GPU Worker | 所有 AI 訓練/推論在此執行 |

### 3-2｜用戶角色說明（7 分鐘）

不同人員與系統的互動方式不同：

| 角色 | 日常操作 | 主要介面 |
|------|----------|----------|
| **系統管理員** | 管理節點、帳號、GPU 配額、系統升級 | Run:ai UI + kubectl CLI |
| **部門主管** | 查看部門 GPU 使用報告、設定專案配額 | Run:ai UI（Analytics 頁面）|
| **研究員** | 提交訓練工作、查看狀態、取得結果 | Run:ai UI + runai CLI |
| **ML Ops** | 部署推論服務、監控服務狀態 | Run:ai UI + API |

---

## 休息（01:10 – 01:15）｜5 分鐘

---

## 模組四：常用操作 Demo（01:15 – 01:45）｜30 分鐘

> 本模組以 Run:ai UI 實際操作示範為主，搭配 runai CLI 介紹

### 4-1｜登入與系統概覽（5 分鐘）

**Demo 步驟：**
1. 開啟瀏覽器，輸入 `https://runai.<客戶網域>`
2. 登入管理員帳號
3. 瀏覽 Overview 頁面，說明各區塊意涵：

| UI 區塊 | 顯示內容 | 說明 |
|---------|----------|------|
| Overview | GPU 使用率、工作負載數量 | 最常看的首頁 |
| Nodes | 每台實體節點的 GPU 狀態 | 確認 XE9780 健康狀況 |
| Workloads | 所有正在執行/排隊的工作 | 看誰在用 GPU |
| Analytics | 歷史使用率趨勢圖 | 月底報告用 |
| Projects | 各專案配額與使用量 | 管理員設定用 |

### 4-2｜建立 Department 與 Project（5 分鐘）

**Demo 步驟：**
1. Resources → Departments → New Department
   - Name：`研究部`
   - GPU Quota：`4`（依實際 GPU 數調整）
2. Resources → Projects → New Project
   - Name：`NLP-實驗`
   - Department：`研究部`
   - GPU Quota：`2`

**重點說明**：
- 配額是「上限」，不保留；有空閒時可借用其他 Project 的配額
- GPU 分數：Quota 可設 `0.5` = 半張 GPU

### 4-3｜建立用戶與分配權限（3 分鐘）

**Demo 步驟：**
1. Settings → Users → Invite User
2. 填入 Email、選擇角色（Researcher）
3. 進入 NLP-實驗 Project → Access Control → 加入用戶

### 4-4｜提交 GPU 工作負載（10 分鐘）

#### 方式 A：透過 Run:ai UI

1. Workloads → New Workload → Training
2. 填入：
   - **名稱**：`my-first-training`
   - **Project**：`NLP-實驗`
   - **Image**：`nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04`
   - **GPU 數量**：`1`
3. 送出後切換到 Workloads 頁面查看狀態
4. 說明狀態流轉：`Pending` → `Running` → `Completed`

#### 方式 B：透過 runai CLI（研究員常用）

```bash
# 登入
runai login --url https://runai.<DOMAIN>

# 設定預設 Project
runai config project NLP-實驗

# 提交工作
runai submit my-training \
  --image nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04 \
  --gpu 1

# 查看工作狀態
runai list jobs

# 查看執行 Log
runai logs my-training

# 刪除工作
runai delete my-training
```

### 4-5｜查看 GPU 使用狀況與告警（7 分鐘）

**Demo 步驟：**
1. **Overview 頁面**：即時 GPU 使用率
2. **Analytics 頁面**：
   - GPU 使用率趨勢（過去 7 天）
   - 各 Department/Project 使用比例
   - 工作等待時間統計
3. **Nodes 頁面**：
   - XE9780 每張 GPU 的使用率
   - 溫度與健康狀態（DCGM 指標）
4. 說明系統內建告警規則（自動觸發，無需額外設定）：
   - GPU Pod 重啟超過 2 次 → Warning
   - 記憶體使用率 > 90% → Critical
   - Cluster 與 Control Plane 斷線 → Critical

---

## 模組五：未來擴展性與演進路徑（01:45 – 01:55）｜10 分鐘

### 5-1｜現有架構擴展方向（5 分鐘）

客戶目前架構（Phase 1）具備良好的擴展基礎：

```
現況（Phase 1）
─────────────────────────────────────
  Run:ai Control Plane（R670 × 2）
         ↓ 管理
  GPU Cluster（XE9780 × 1）
  └── GPU 工作負載

未來擴展（Phase 2）
─────────────────────────────────────
  Run:ai Control Plane（R670 × 2）← 不需變動
         ↓ 管理
  GPU Cluster A（XE9780 × 1）← 現有
  GPU Cluster B（新增 GPU 伺服器）← 直接加入，Control Plane 統一管理
  GPU Cluster C（遠端機房）← 跨機房多叢集
```

| 擴展場景 | 作法 | 影響範圍 |
|----------|------|----------|
| 增加 GPU 數量 | XE9780 加裝 GPU 卡（硬體擴充）| 不需重新安裝軟體 |
| 增加 GPU 節點 | 新增一台 GPU 伺服器加入 K8s | K8s 加節點，Run:ai 自動識別 |
| 多機房管理 | 新機房部署 Run:ai Cluster，連回同一 Control Plane | R670 Control Plane 不需變動 |
| 雲端混合 | 將 EKS/GKE 上的 GPU 納管 | 同一 UI 統一管理地端+雲端 |

### 5-2｜單一 Control Plane 管理多叢集（5 分鐘）

```
                Run:ai Control Plane（R670 × 2）
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
  本地 Cluster A  本地 Cluster B  遠端 Cluster C
  XE9780 × 1     XE9780 × 3     雲端 A100 × 8
  （現有）        （未來擴充）    （雲端資源）
```

**關鍵優勢**：
- **統一 UI**：所有 GPU 資源在同一個 Overview 頁面
- **統一配額**：跨叢集的 Department/Project 配額設定
- **統一帳號**：研究員不需知道工作跑在哪個叢集
- **Control Plane 不需更動**：現有 R670 × 2 可持續服務，只需在新節點部署 Cluster 元件

**擴展建議**（依需求演進）：

```
現在              → 未來 6–12 個月      → 未來 1–2 年
────────────────────────────────────────────────────
XE9780 × 1        → + XE9780 × N        → + 多機房 / 雲端混合
單一 GPU 池        → 多 Node Pool 分組   → 跨站點統一排程
本地帳號          → 整合 AD/SSO         → 細粒度 RBAC
```

---

## Q&A 與總結（01:55 – 02:00）｜5 分鐘

### 重點複習

| 問題 | 答案 |
|------|------|
| Run:ai 住在哪裡？ | Control Plane 在 R670，Cluster 排程器在 XE9780 |
| 研究員怎麼使用？ | Web UI 提交工作，或用 runai CLI |
| 怎麼控制誰能用多少 GPU？ | Department → Project → GPU Quota 三層設定 |
| 未來想加 GPU 怎麼辦？ | 加節點加入 K8s，Run:ai 自動識別，Control Plane 不需動 |
| 資料安全嗎？ | Cluster 只傳 Metadata，用戶資料/模型不離開本地 |

### 學員帶走的三件事

1. **Run:ai = GPU 的交通管理系統**：讓稀缺的 GPU 資源最大化利用
2. **R670 管理、XE9780 執行**：兩台角色清楚，擴充只需加 XE9780 數量
3. **從 UI 就能完成日常操作**：提交工作、查看狀態、管理配額，不需 kubectl 指令

---

## 附件：訓練前建議準備事項

### 講師準備

- [ ] 確認 Run:ai UI 可正常登入（`https://runai.<DOMAIN>`）
- [ ] 準備示範用帳號（Admin + Researcher 各一組）
- [ ] 預先建立 Demo 用 Department/Project
- [ ] 確認 XE9780 GPU 正常，可執行 CUDA 測試工作負載
- [ ] 備妥螢幕投影設備（建議 1920×1080 以上）

### 學員準備

- [ ] 攜帶筆電（可選，用於跟著操作 runai CLI）
- [ ] 確認可連線至客戶內部網路

### 建議補充教材

| 資源 | 說明 |
|------|------|
| Run:ai 官方文件 | https://run-ai-docs.nvidia.com/ |
| runai CLI Cheat Sheet | 常用指令速查（可另行製作一頁版） |
| GPU 使用率查看 SOP | 月報用，Analytics 頁面截圖教學 |
