# NVIDIA & Dell AI Factory — 官方文件庫

整理日期：2026-05-24 | 版本基準：最新 Active 版本 | 語言：英文原文

---

## 建議閱讀路徑（初次接觸）

**Step 1 — 建立概念**
→ 04, 07, 10

**Step 2 — 了解 Dell AI Factory 架構**
→ 01, 08, 09, 11, 15

**Step 3 — NVAIE 軟體平台**
→ 02, 03

**Step 4 — Kubernetes / Run:ai 部署**
→ 21, 17, 05

**Step 5 — 進階主題**
→ 13, 12, 19, 20, 06, 18

---

## 文件清單

### NVIDIA Enterprise Reference Architectures

| # | 檔名 | 說明 | 來源 |
|---|------|------|------|
| 01 | `01-NVIDIA-Enterprise-Reference-Architecture-Whitepaper-2025.pdf` | NVIDIA ERA 總覽白皮書 (May 2025)；AI Factory 概念、架構組成、各 RA 說明 | [docs.nvidia.com](https://docs.nvidia.com/enterprise-reference-architectures/white-paper.pdf) |
| 17 | `17-NVIDIA-ERA-NIM-LLM-RunAI-Vanilla-Kubernetes.pdf` | **Run:ai + Vanilla Kubernetes 參考架構**；NIM LLM 推論、GPU 排程、Run:ai 整合 | [docs.nvidia.com](https://docs.nvidia.com/enterprise-reference-architectures/nim-llm-with-run-ai-and-vanilla-kubernetes.pdf) |
| 18 | `18-NVIDIA-ERA-AI-Q-Research-Agent-Blueprint.pdf` | AI-Q Research Agent Blueprint；Agentic RAG + NIM + Run:ai 完整架構 | [docs.nvidia.com](https://docs.nvidia.com/enterprise-reference-architectures/ai-q-research-agent-blueprint.pdf) |
| 19 | `19-NVIDIA-ERA-Observability-Guide.pdf` | AI Factory 可觀測性指南；Prometheus/Grafana 監控 GPU 叢集 | [docs.nvidia.com](https://docs.nvidia.com/enterprise-reference-architectures/observability-guide.pdf) |
| 20 | `20-NVIDIA-ERA-Base-Command-Manager-Deployment-Guide.pdf` | Base Command Manager 部署指南；多叢集管理、作業排程 | [docs.nvidia.com](https://docs.nvidia.com/enterprise-reference-architectures/base-command-manager-deployment-guide.pdf) |
| 21 | `21-NVIDIA-ERA-Upstream-Kubernetes-Deployment-Guide.pdf` | Upstream Kubernetes 部署指南；裸機 K8s + GPU Operator 完整流程 | [docs.nvidia.com](https://docs.nvidia.com/enterprise-reference-architectures/upstream-kubernetes-deployment-guide.pdf) |

### NVIDIA AI Enterprise (NVAIE)

| # | 檔名 | 說明 | 來源 |
|---|------|------|------|
| 02 | `02-NVIDIA-AI-Enterprise-5.1-User-Guide.pdf` | NVAIE 5.1 完整使用者指南；安裝、授權、vGPU/Passthrough、Kubernetes | [docs.nvidia.com](https://docs.nvidia.com/ai-enterprise/5.1/pdf/nvidia-ai-enterprise-user-guide.pdf) |
| 03 | `03-NVIDIA-AI-Enterprise-5.1-Quick-Start-Guide.pdf` | NVAIE 5.1 快速入門指南；最短路徑完成基礎安裝 | [docs.nvidia.com](https://docs.nvidia.com/ai-enterprise/5.1/pdf/nvidia-ai-enterprise-quick-start-guide.pdf) |
| 04 | `04-NVIDIA-AI-Enterprise-Solution-Overview.pdf` | NVAIE Solution Overview；軟體堆疊、元件說明、商業價值 | [images.nvidia.com](https://images.nvidia.com/content/APAC/assets/in/nvaie-solution-overview-4-0-update-2817769.pdf) |
| 06 | `06-NVIDIA-Secure-AI-Blackwell-Hopper-Whitepaper.pdf` | NVIDIA Secure AI 白皮書 (Aug 2025)；Blackwell/Hopper 安全功能、機密運算 | [docs.nvidia.com](https://docs.nvidia.com/nvidia-secure-ai-with-blackwell-and-hopper-gpus-whitepaper.pdf) |
| 07 | `07-NVIDIA-AI-Enterprise-IDC-Infobrief.pdf` | IDC Infobrief：NVAIE 商業投資報酬率分析 | [images.nvidia.com](https://images.nvidia.com/aem-dam/Solutions/documents/ai-enterprise-idc-infobrief.pdf) |

### NVIDIA Networking & Systems

| # | 檔名 | 說明 | 來源 |
|---|------|------|------|
| 05 | `05-NVIDIA-Network-Operator-v25.7-Kubernetes.pdf` | NVIDIA Network Operator v25.7；Kubernetes 網路加速、InfiniBand/RoCE | [docs.nvidia.com](https://docs.nvidia.com/networking/display/kubernetes2570/nvidia-network-operator-v25-7-0.0.pdf) |
| 16 | `16-NVIDIA-DGX-Systems-Solution-Brief.pdf` | DGX Systems Solution Brief；DGX H100/H200 規格與部署情境 | [images.nvidia.com](https://images.nvidia.com/aem-dam/Solutions/Data-Center/dgx-systems/dgx-systems-solution-brief.pdf) |

### Dell AI Factory with NVIDIA

| # | 檔名 | 說明 | 來源 |
|---|------|------|------|
| 08 | `08-Dell-AI-Factory-NVIDIA-ERA-2-8-5-200-Config-Brief.pdf` | Dell AI Factory ERA 認證配置 2-8-5-200 (H100 SXM)；驗證架構說明 | [delltechnologies.com](https://www.delltechnologies.com/asset/en-us/solutions/infrastructure-solutions/briefs-summaries/nvidia-2-8-5-200-era-configuration-endorsed-for-the-dell-ai-factory-with-nvidia-brief.pdf) |
| 09 | `09-Dell-AI-Factory-NVIDIA-ERA-2-8-9-400-Config-Brief.pdf` | Dell AI Factory ERA 認證配置 2-8-9-400 (H200 SXM)；XE9680 大型訓練叢集 | [delltechnologies.com](https://www.delltechnologies.com/asset/en-us/solutions/infrastructure-solutions/briefs-summaries/nvidia-2-8-9-400-configuration-era-endorsed-for-the-dell-ai-factory-with-nvidia-brief.pdf) |
| 10 | `10-Dell-AI-Factory-NVIDIA-eBook.pdf` | Dell AI Factory with NVIDIA eBook；端對端解決方案全覽 | [delltechnologies.com](https://www.delltechnologies.com/asset/en-us/solutions/business-solutions/briefs-summaries/dell-ai-factory-with-nvidia-ebook.pdf) |
| 11 | `11-Dell-EMC-Reference-Architecture-AI-Deep-Learning-NVIDIA.pdf` | Dell EMC Ready Solutions for AI — Deep Learning 參考架構 | [dl.dell.com](https://dl.dell.com/manuals/common/dellemc_readysol_ai_deeplearning_nvidia.pdf) |
| 13 | `13-Dell-Generative-AI-Inferencing-Design-Guide.pdf` | Dell Validated Design — Generative AI Inferencing 設計指南；LLM 推論架構 | [delltechnologies.com](https://www.delltechnologies.com/asset/en-gb/solutions/business-solutions/technical-support/h19686-gen-ai-inferencing-dg.pdf) |
| 12 | `12-Dell-Scalable-Architecture-RAG-NVIDIA-Microservices.pdf` | Dell Scalable Architecture for RAG with NVIDIA Microservices 白皮書 | [infohub.delltechnologies.com](https://infohub.delltechnologies.com/static/media/client/7phukh/DAM_88745fcb-8e2b-46ce-9680-284da15ce94a.pdf) |
| 15 | `15-Dell-PowerEdge-AI-Acceleration-Innovate-Faster.pdf` | Dell PowerEdge AI Acceleration Solution Brief；伺服器 AI 加速技術概覽 | [delltechnologies.com](https://www.delltechnologies.com/asset/en-us/products/servers/briefs-summaries/poweredge-acceleration-innovate-faster-for-ai.pdf) |
| 14 | `14-Dell-PowerEdge-XE9680-Spec-Sheet.pdf` | Dell PowerEdge XE9680 規格表；8-GPU flagship 伺服器硬體規格 | [delltechnologies.com](https://www.delltechnologies.com/asset/en-in/products/servers/technical-support/poweredge-xe9680-spec-sheet.pdf) |

---

## 線上補充資源（無 PDF 版本）

以下官方文件為 HTML 格式，建議搭配上述 PDF 一起參考：

| 主題 | URL |
|------|-----|
| NVIDIA Run:ai Self-Hosted 安裝 | https://run-ai-docs.nvidia.com/self-hosted/getting-started/ |
| NVIDIA AI Enterprise AI Factory Design Guide | https://docs.nvidia.com/ai-enterprise/planning-resource/ai-factory-white-paper/latest/ |
| NVIDIA GPU Operator Getting Started | https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html |
| NVAIE Red Hat AI Factory 部署指南 | https://docs.nvidia.com/ai-enterprise/deployment/red-hat-ai-factory/latest/ |
| NVAIE OpenShift on Bare Metal 部署指南 | https://docs.nvidia.com/ai-enterprise/deployment/openshift-on-bare-metal/latest/ |
| NVIDIA Enterprise Reference Architectures 索引 | https://docs.nvidia.com/enterprise-reference-architectures/index.html |
