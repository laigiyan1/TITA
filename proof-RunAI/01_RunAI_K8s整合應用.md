# NVIDIA Run.ai 與 Kubernetes 容器平台整合應用

## 1. 概述

NVIDIA Run.ai 是一套建立於 Kubernetes 之上的 AI 基礎設施管理平台，提供 GPU 資源的虛擬化、排程與共享能力，讓多個工作負載得以高效共用有限的 GPU 資源，大幅提升 GPU 使用率並降低閒置浪費。

---

## 2. 核心架構

### 2.1 Run.ai 與 K8s 關係

```
┌───────────────────────────────────────────────────────┐
│                   Run.ai 管理平面                      │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Run.ai UI  │  │  Run.ai CLI  │  │  REST API    │  │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  │
│         └────────────────┴─────────────────┘           │
│                          │                             │
│              ┌───────────▼──────────┐                  │
│              │   Run.ai Scheduler   │                  │
│              │  (K8s Scheduler 擴充) │                  │
│              └───────────┬──────────┘                  │
└──────────────────────────┼──────────────────────────── ┘
                           │
┌──────────────────────────▼──────────────────────────── ┐
│                 Kubernetes 叢集                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Node 1 (GPU×8)   Node 2 (GPU×8)   Node 3 (GPU×8) │  │
│  │  NVIDIA Device Plugin + Run.ai Agent              │  │
│  └──────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

### 2.2 關鍵元件說明

| 元件 | 層次 | 功能 |
|------|------|------|
| Run.ai Control Plane | 管理層 | 統一入口、排程決策、使用率儀表板 |
| Run.ai Scheduler | K8s 擴充 | 替換或擴充原生 kube-scheduler，支援 GPU fraction |
| Run.ai Agent (DaemonSet) | 節點層 | 回報 GPU 狀態、執行資源隔離 |
| NVIDIA Device Plugin | 節點層 | 向 K8s 暴露 GPU 資源 |
| DCGM Exporter | 監控層 | 提供 GPU 遙測指標給 Prometheus |

---

## 3. GPU 資源管理模式

### 3.1 GPU 分數 (GPU Fraction)

Run.ai 允許將單張實體 GPU 分割給多個工作負載共用。v2.25 透過 **Resource Interface** 管理標準 Kubernetes workload，以 Kubeflow PyTorchJob 為例：

```yaml
# v2.25 寫法：標準 Kubeflow PyTorchJob + Run.ai Scheduler
# 舊版 run.ai/v2alpha1 TrainingWorkload CRD 已不建議使用
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: train-resnet
  namespace: research-team
  # ⚠️ GPU fraction annotation 需放在 Pod template 層級（非 workload 頂層），
  #    以確保 Run.ai Scheduler 能正確辨識
spec:
  pytorchReplicaSpecs:
    Worker:
      replicas: 1
      template:
        metadata:
          annotations:
            gpu-fraction: "0.5"          # Run.ai GPU Fraction（Pod 層級，無 runai/ prefix）
            # 若使用 Dynamic GPU Fractions 限制記憶體上限，可額外加：
            # gpu-memory: "20G"          # 以 GPU 記憶體量指定分數（擇一使用）
        spec:
          schedulerName: runai-scheduler   # 指定 Run.ai Scheduler
          containers:
            - name: pytorch
              image: nvcr.io/nvidia/pytorch:24.04-py3
              command: ["python", "train.py"]
              resources:
                limits:
                  nvidia.com/gpu: "1"      # GPU slot 佔位，實際分數由 fraction 管理
              # 若透過環境變數限制 GPU 記憶體（Dynamic GPU Fractions）：
              # env:
              #   - name: RUNAI_GPU_MEMORY_LIMIT
              #     value: "20Gi"
```

### 3.2 動態 GPU 記憶體分配 (Dynamic GPU Memory)

- **MIG (Multi-Instance GPU)**：硬體層隔離，適合 A100/H100，提供確定性效能保證
- **Time-Slicing**：軟體層分時，適合較舊 GPU，延遲較高但相容性廣
- **Dynamic GPU Fractions**：Run.ai 動態分割 GPU 記憶體，允許多個工作負載共用同一顆 GPU，支援 0.25 / 0.5 等分數申請（注意：NVIDIA vGPU 在 Run.ai v2.25 中**不支援**，亦請勿與 NVIDIA 原廠 vGPU 產品混淆）

| 方式 | 隔離等級 | 效能影響 | 適用情境 |
|------|----------|----------|----------|
| MIG | 硬體 | 最低 | 推論服務、SLA 要求嚴格 |
| Time-Slicing | 軟體 | 中等 | 開發/測試環境 |
| Dynamic GPU Fractions | 軟體 | 低～中 | 訓練批次作業、小模型推論 |

### 3.3 排程策略

- **Bin Packing**：優先填滿現有節點，減少碎片化
- **Spread**：分散至不同節點，提升容錯能力
- **Preemption（搶佔）**：低優先權作業可被高優先權作業暫停並回復

---

## 4. 專案 (Project) 與部門 (Department) 資源配額

```
Organization
 └── Department: AI Research        (GPU Quota: 40)
      ├── Project: NLP-Team          (GPU Quota: 16)
      ├── Project: CV-Team           (GPU Quota: 16)
      └── Project: RL-Team           (GPU Quota: 8)
 └── Department: AI Platform        (GPU Quota: 20)
      ├── Project: MLOps             (GPU Quota: 10)
      └── Project: Inference-Prod    (GPU Quota: 10)
```

- 每個 Project 對應一個 K8s Namespace
- 超出配額的作業進入排隊佇列，不佔用資源
- 支援 **Over-Quota**（借用其他 Project 閒置配額）

---

## 5. 工作負載類型支援

v2.25 透過 **Resource Interface** 支援以下標準 workload 類型，舊版專有 CRD（`run.ai/v2alpha1 TrainingWorkload / DistributedWorkload / InferenceWorkload`）已不建議於新部署中使用。

| 類別 | 支援框架 / 資源 | 適用場景 |
|------|----------------|----------|
| 一般訓練 | Kubernetes `Job` | 單節點批次訓練、腳本執行 |
| 互動開發 | Kubernetes `Pod` / `Deployment` | Jupyter Notebook、開發除錯 |
| 分散式訓練 | Kubeflow `PyTorchJob` / `TFJob` / `MPIJob` | 多節點 PyTorch DDP、Horovod |
| 大規模分散式 | Ray `RayJob` / `RayCluster` | 大規模分散式訓練與超參數搜尋 |
| 推論服務 | KServe `InferenceService` | 標準模型推論（含自動擴縮） |
| NVIDIA 推論 | NIM（NVIDIA Inference Microservices） | 最佳化的 NVIDIA 模型推論部署 |

---

## 6. 整合元件清單

| 整合目標 | 整合方式 | 說明 |
|----------|----------|------|
| Kubernetes | Native | Run.ai 建立於 K8s 之上，使用 CRD 擴充 |
| NVIDIA GPU Operator | Helm Chart | 自動安裝驅動、Device Plugin、DCGM |
| Prometheus / Grafana | ServiceMonitor | GPU/作業指標視覺化 |
| MLflow / W&B | SDK 整合 | 實驗追蹤與指標紀錄 |
| Jupyter Hub | SSO + PVC | 互動式開發環境 |
| Harbor / Registry | ImagePullSecret | 私有映像檔倉庫 |
| LDAP / AD | OIDC / SAML | 使用者身分驗證與授權 |
| S3 / NFS / Ceph | PVC / CSI Driver | 訓練資料與模型儲存 |

---

## 7. 網路架構考量

- **Ingress Controller**：Nginx 或 Traefik，提供 Run.ai UI 及 API 入口
- **GPU Direct RDMA**：啟用 InfiniBand / RoCE 以支援高速節點間 GPU 通訊（分散式訓練必要）
- **Network Policy**：限制 Namespace 間流量，隔離研究與正式環境

---

## 8. 安全性設計

- RBAC：透過 K8s RBAC 綁定 Run.ai Role（`researcherRole`、`mlEngineerRole`、`adminRole`）
- Pod Security Admission：限制特權容器，避免安全風險
- 映像檔掃描：整合 Trivy / Clair 於 CI/CD 流程
- 審計日誌：K8s Audit Log + Run.ai 操作日誌集中至 SIEM

---

## 9. 效益摘要

| 指標 | 導入前 | 導入後（預期） |
|------|--------|---------------|
| GPU 平均使用率 | 30–50% | 70–85% |
| 作業等待時間 | 無排隊機制 | 可預期佇列 |
| 資源分配透明度 | 無 | 即時儀表板 |
| 多租戶隔離 | 手動分配 | 配額自動管理 |
