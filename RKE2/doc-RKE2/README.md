# RKE2 技術文件庫

> 整理日期：2026-05-26
> 目標版本：**RKE2 v1.35.5+rke2r1（stable channel，本文件主軸）**；參考列出 latest channel v1.36.1+rke2r1
> 主要參考來源：[https://docs.rke2.io/](https://docs.rke2.io/)

本文件庫專為**初次接觸 RKE2 的技術人員**撰寫，涵蓋 Ubuntu 24.04 LTS 上部署 Rancher Kubernetes Engine 2（RKE2）並建構為 NVIDIA Run:ai 系統執行平台的完整流程。

---

## 1. 建置情境總覽

| 角色 | 數量 | 平台 | 用途 |
|------|------|------|------|
| RKE2 Server Node（Control-Plane + etcd） | 3 | VMware vSphere（Dell PowerEdge R670 × 2 構成） | 高可用控制平面 |
| RKE2 Worker Node（CPU） | 2 | VMware vSphere | 一般工作負載 |
| RKE2 Worker Node（GPU） | 1 | Dell PowerEdge XE9780（HGX 平台 8-GPU 機架；GPU 型號依採購決定，**待驗證**） | Run:ai 推論/訓練工作負載 |

> **重要**：HGX 實體機**不要**虛擬化（避免 vGPU 與 RKE2 + GPU Operator 之間額外的整合複雜度）。建議在 HGX 上直接安裝 Ubuntu 24.04 LTS 並以 Bare-metal 形式加入 RKE2 叢集。詳見 [11-RKE2-VM-Baremetal-Hybrid-Notes.md](11-RKE2-VM-Baremetal-Hybrid-Notes.md)。

---

## 2. 文件清單

| # | 文件 | 內容說明 |
|---|------|----------|
| 00 | [Overview-and-Architecture](00-RKE2-Overview-and-Architecture.md) | RKE2 產品定位、核心元件、Server/Agent 架構、與 K3s/RKE1 差異 |
| 01 | [Prerequisites](01-RKE2-Prerequisites.md) | Ubuntu 24.04 OS 前置設定、硬體規格、網路埠、防火牆／NetworkManager／AppArmor 處理 |
| 02 | [Installation-HA](02-RKE2-Installation-HA.md) | 3 節點 Control-Plane HA 部署完整流程、固定註冊地址設定 |
| 03 | [Worker-Node-Setup](03-RKE2-Worker-Node-Setup.md) | VM Worker 與 GPU Bare-metal Worker 加入叢集的操作步驟 |
| 04 | [Configuration](04-RKE2-Configuration.md) | `config.yaml` 完整參數、CNI 切換、disable 內建元件、registries.yaml |
| 05 | [Validation](05-RKE2-Validation.md) | 叢集健康檢查、Pod 部署測試、GPU 資源驗證 |
| 06 | [Backup-and-Restore](06-RKE2-Backup-and-Restore.md) | etcd 自動／手動快照、S3 備份、單節點與多節點災難還原 |
| 07 | [Upgrade](07-RKE2-Upgrade.md) | 手動升級與 system-upgrade-controller 自動升級、回滾流程 |
| 08 | [Monitoring-and-Logging](08-RKE2-Monitoring-and-Logging.md) | journalctl、Pod log、metrics-server、Prometheus 整合切入點 |
| 09 | [Troubleshooting](09-RKE2-Troubleshooting.md) | 啟動失敗、節點無法 join、CNI 異常、常見問題排查 |
| 10 | [Deployment-Checklist](10-RKE2-Deployment-Checklist.md) | 部署前到部署後的逐項檢查清單 |
| 11 | [VM-Baremetal-Hybrid-Notes](11-RKE2-VM-Baremetal-Hybrid-Notes.md) | VMware VM 與 HGX 實體機混合節點的維運注意事項 |

---

## 3. 建議閱讀路徑

### 路徑 A：初次部署者（從零開始）
00 → 01 → 02 → 03 → 04 → 05 → 11 → 10

### 路徑 B：架構審查者（評估技術選型）
00 → 01 → 11 → 06 → 07

### 路徑 C：維運交接（接手已建置叢集）
00 → 04 → 06 → 07 → 08 → 09 → 11

---

## 4. 版本選擇決策指引

RKE2 採用與上游 Kubernetes 同步的版本策略。Channel 對應關係如下（資料來源：[https://update.rke2.io/v1-release/channels](https://update.rke2.io/v1-release/channels)，2026-05-26 查詢）：

| Channel | 版本 | 適用情境 |
|---------|------|----------|
| `stable` | v1.35.5+rke2r1 | **本文件預設選擇**。生產環境首選，已通過社群充分驗證 |
| `latest` | v1.36.1+rke2r1 | 需要最新功能（例如 Traefik 預設、containerd 2.2.x）；接受可能尚未完全成熟 |
| `v1.34` | v1.34.8+rke2r1 | 仍處於支援週期內的前一代次版本 |

**Run:ai 相容性決策**：依 NVIDIA Run:ai Self-hosted v2.25 官方文件（[support matrix](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/support-matrix.md)、[system requirements](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/install-using-helm/system-requirements.md)，2026-05-26 查詢）：

- **Run:ai 2.25 / 2.24** 支援 Kubernetes **1.33 – 1.35**
- 因此本文件**建議採用 RKE2 `stable` channel（v1.35.5+rke2r1）**，對應 K8s 1.35，與 Run:ai 2.24/2.25 完全相容
- **不建議**直接使用 latest channel 的 v1.36.x，會超出 Run:ai 2.25 的官方支援範圍（**部署前請至 Run:ai Self-hosted v2.25 [support matrix](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/support-matrix.md) 確認最新版本**）

---

## 5. 主要參考 URL 速查

| 主題 | URL |
|------|-----|
| RKE2 官方文件首頁 | [https://docs.rke2.io/](https://docs.rke2.io/) |
| 架構說明 | [https://docs.rke2.io/architecture](https://docs.rke2.io/architecture) |
| 系統需求 | [https://docs.rke2.io/install/requirements](https://docs.rke2.io/install/requirements) |
| HA 部署 | [https://docs.rke2.io/install/ha](https://docs.rke2.io/install/ha) |
| 備份與還原 | [https://docs.rke2.io/datastore/backup_restore](https://docs.rke2.io/datastore/backup_restore) |
| 手動升級 | [https://docs.rke2.io/upgrades/manual](https://docs.rke2.io/upgrades/manual) |
| 回滾 | [https://docs.rke2.io/upgrades/roll-back](https://docs.rke2.io/upgrades/roll-back) |
| GPU Operator 整合 | [https://docs.rke2.io/add-ons/gpu_operators](https://docs.rke2.io/add-ons/gpu_operators) |
| 已知問題 | [https://docs.rke2.io/known_issues](https://docs.rke2.io/known_issues) |
| 進階設定 | [https://docs.rke2.io/advanced](https://docs.rke2.io/advanced) |
| Release Channels | [https://update.rke2.io/v1-release/channels](https://update.rke2.io/v1-release/channels) |
| GitHub Releases | [https://github.com/rancher/rke2/releases](https://github.com/rancher/rke2/releases) |
| SUSE 鏡像版本 | [https://documentation.suse.com/cloudnative/rke2/latest/en/](https://documentation.suse.com/cloudnative/rke2/latest/en/) |
| Run:ai Self-hosted v2.25 support matrix | [https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/support-matrix.md](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/support-matrix.md) |
| Run:ai Self-hosted v2.25 system requirements | [https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/install-using-helm/system-requirements.md](https://run-ai-docs.nvidia.com/self-hosted/2.25/getting-started/installation/install-using-helm/system-requirements.md) |

---

## 6. 圖片清單（`pic-RKE2/`）

| 檔名 | 用途 | 來源 |
|------|------|------|
| `rke2-architecture-overview.png` | RKE2 整體架構圖（Server + Agent + 元件） | [docs.rke2.io 架構頁面](https://docs.rke2.io/assets/images/overview-06f8a098e271952bfe5db78b3a0e9b25.png) |
| `rke2-ha-topology.svg` | RKE2 高可用拓撲圖（3 Server + N Agent + Fixed Registration Address） | [docs.rke2.io HA 安裝頁面](https://docs.rke2.io/assets/images/rke2-production-setup-f5158274308e4a8976ea46273d6cb5c5.svg) |

---

## 7. PDF 清單（`pdf-RKE2/`）

目前 RKE2 官方未發布獨立 PDF 白皮書或 Reference Architecture，所有官方資訊集中於 [docs.rke2.io](https://docs.rke2.io/)。本資料夾保留供後續加入 NVIDIA Run:ai 官方 PDF 文件（如 SaaS Deployment Guide）使用。

---

## 8. 使用本文件的注意事項

1. **所有版本號、容量規格皆有來源 URL**：請以原始 URL 為準，版本可能在 docs.rke2.io 更新後改變。
2. **標示「待驗證」的項目**：表示官方文件未明確列出，僅供參考；正式上線前須以實際環境驗證。
3. **HA 設定中的關鍵參數**：所有 server 節點上必須一致（例如 `cluster-cidr`、`service-cidr`、`cni`、`disable`），詳見 [04-RKE2-Configuration.md](04-RKE2-Configuration.md)。
4. **Token 備份是 etcd 還原的前提**：詳見 [06-RKE2-Backup-and-Restore.md](06-RKE2-Backup-and-Restore.md)，不要只備份 snapshot 檔案。
