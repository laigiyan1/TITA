# NVIDIA AI Workflow 簡述

> 產出日期：2026-05-26  
> 來源：本知識庫 `raw/doc-RunAI/08-NVAIE-Overview.md`、`raw/doc-RunAI/09-NVAIE-RunAI-Integration.md`  
> 類型：AI 整理回答

---

根據本知識庫（NVAIE + Run:AI），NVIDIA 的 AI Workflow 是以 **NVIDIA AI Enterprise (NVAIE)** 為基礎的端對端 AI 生命週期,從原型開發到生產部署皆涵蓋。

## 核心概念:兩層可組合式架構（Composable Stack）

```
┌─────────────────────────────────────────┐
│  Application Layer（應用層）             │
│  NIM Microservices / NeMo / Omniverse  │
│  PyTorch / TensorFlow / Pre-trained    │
│  CUDA-X / Domain SDKs (TAO, Riva...)   │
├─────────────────────────────────────────┤
│  Infrastructure Layer（基礎設施層）       │
│  GPU Driver / GPU Operator / Run:ai    │
│  Container Toolkit / vGPU·MIG / NIM Op │
└─────────────────────────────────────────┘
```

兩層**獨立版本週期**,可分開升級（例如更新 NIM 推論服務不需動到 GPU Driver）。

## 典型 AI Workflow 階段

| 階段 | 對應元件 | 說明 |
|------|---------|------|
| 1. 資料 / 模型準備 | NGC Catalog、Pre-trained Models | 預訓練模型與資料管線 |
| 2. 訓練 / 微調 | NeMo Framework、PyTorch、TAO | LLM / 視覺 / 語音模型開發 |
| 3. 資源排程 | **NVIDIA Run:ai** | GPU 共享、佇列、配額 |
| 4. 推論部署 | NIM Microservices + NIM Operator | 容器化推論服務 |
| 5. 監控 / 治理 | GPU Operator + DCGM | 驅動、健康度、用量 |

## 三大特點

1. **企業級 SLA**：含 Business Standard 支援、Security Patch
2. **跨平台**：Bare Metal、VMware vSphere、AWS/GCP/Azure/OCI 皆可
3. **彈性授權**：年訂閱、雲端消費型、永久授權；H100/H200 購買即附 5 年訂閱

## 與本案 Run:AI 的關係

Run:AI 是 NVAIE 8.1 Infrastructure Layer 的核心元件（版本 2.25），負責在 Kubernetes 上做 **AI 工作負載排程與 GPU 資源管理**，把 NVAIE 軟體棧的訓練 / 推論工作流程「分配到正確的 GPU」上，是整個 AI Workflow 的調度中樞。

---

## 延伸閱讀

- [[08-NVAIE-Overview]] — NVAIE 平台概覽與架構
- [[09-NVAIE-RunAI-Integration]] — NVAIE × Run:AI 整合說明
- [NVAIE 產品概覽（官方）](https://docs.nvidia.com/ai-enterprise/latest/product-overview/index.html)
- [NVAIE 8.1 Release Notes](https://docs.nvidia.com/ai-enterprise/release-8/8.1/index.html)
