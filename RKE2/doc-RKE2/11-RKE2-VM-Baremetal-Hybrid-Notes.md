# 11 – VM 與實體機混合節點維運注意事項

> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）
> 適用情境：本案的 3 Server (VM) + 2 CPU Worker (VM) + 1 GPU Worker (HGX Bare-metal) 拓撲

> **文件性質聲明**：本文件**非官方文件直接引用**，為**實務指引**整理，目的在補充 RKE2/Run:ai 官方文件未直接涵蓋的混合部署注意事項。每一項建議皆為「需依組織實際環境調整」的判斷依據，**並非單一正確答案**。標示「**待驗證**」者請務必在 PoC 階段驗證後採用。組織內部若有與本文件衝突的標準，以組織標準為準。

---

## 1. 為何 GPU Worker 不建議虛擬化

| 因素 | 說明 |
|------|------|
| **vGPU 授權** | NVIDIA vGPU 需要額外授權（NVIDIA AI Enterprise / vGPU 授權）；單純 PCI Passthrough 不算 vGPU |
| **HGX 拓撲完整性** | HGX 板載 NVLink / NVSwitch 拓撲在虛擬化後不易完整透出，影響多 GPU 訓練性能 |
| **vMotion 不可用** | GPU passthrough VM 無法 vMotion，可用性優勢消失 |
| **整合複雜度** | RKE2 + GPU Operator + vGPU host driver 增加版本對齊困難 |
| **Run:ai 場景** | Run:ai 大量假設 GPU 為實體 device，部分 advanced scheduling（fractional GPU、MIG）在 vGPU 上行為不同 |

**結論**：HGX 直接安裝 Ubuntu Server + RKE2 agent，GPU 由 GPU Operator 管理 → 最簡且性能最佳。

---

## 2. VM 與實體機共存的差異點

### 2.1 CPU / Memory 規格不對稱

| 角色 | vCPU | RAM | NUMA |
|------|------|-----|------|
| VM Server | 8 vCPU | 16 GB | 通常單一 vNUMA |
| VM CPU Worker | 8 vCPU | 32 GB | 通常單一 vNUMA |
| HGX GPU Worker | 通常 100+ 物理核心 | 通常 1 TB+ | 多 NUMA node |

**影響**：
- HGX 節點上的 Pod 若未設定 `topologyManagerPolicy`，可能因 NUMA 分配不當影響 GPU 訓練效能
- HGX 節點的可分配 CPU/RAM 數量遠大於 VM，預設 scheduler 可能將過多 system pod 排到 HGX 上 → 應透過 [03 §3.2 的 taint](03-RKE2-Worker-Node-Setup.md) 阻擋

### 2.2 網路 MTU 差異

- VMware vSwitch / NSX-T overlay 通常 MTU 9000（jumbo frame）
- HGX 實體機網卡通常也支援 jumbo frame（25/100 GbE）
- **CNI 設定的 MTU 必須一致**：Canal 預設 VXLAN MTU 1450（從 1500 - VXLAN overhead）
- **若實體與虛擬網段間有不同 MTU**，會出現「小封包通、大封包失敗」這類難以察覺的現象

> **待驗證**：本案實體網段與 vSphere 端的 MTU 設定，務必協調網路工程師統一。RKE2 / Canal 的 `veth-mtu` 可透過 HelmChartConfig 調整（[04 §5](04-RKE2-Configuration.md)）。

### 2.3 儲存層延遲

- VMware VM 的 etcd VMDK：若 datastore 為 vSAN / shared 儲存，fsync 延遲依底層硬體可能 5–20 ms
- HGX 實體機本地 NVMe：fsync 通常 < 1 ms

**影響**：
- etcd 只在 server 節點上，**HGX 不放 etcd**，但 worker 上的 emptyDir / hostPath 寫入差異仍會反映在工作負載性能上
- 工作負載寫入大量資料時建議使用 GPU 節點本地 NVMe（hostPath 或 local-storage CSI）

### 2.4 NIC 與網卡綁定

- VM：單一 VMXNET3
- HGX：多 NIC（管理網 + 高速網 + 可能 IB）

**RKE2 預設綁定**：
- 預設使用 `INTERNAL-IP`（第一個非 loopback 介面）
- 若有多 NIC，於 config.yaml 明確指定：
  ```yaml
  node-ip: 10.10.10.100        # 控制平面流量介面
  node-external-ip: 192.168.10.100   # 外部訪問介面
  ```

---

## 3. 跨平台維運：常見差異

| 操作 | VM 節點 | HGX 實體節點 |
|------|---------|--------------|
| 重啟 | vSphere 介面 reboot；數十秒可回 | 實體 BMC（iDRAC）重啟；3-5 分鐘 POST + 開機 |
| 替換故障節點 | 部署新 VM；數分鐘 | 硬體採購/送修；數天-數週 |
| 硬體擴充 | vCPU/RAM 線上熱新增（需 OS 支援） | 停機加實體卡 |
| 韌體更新 | vSphere 主機端負責 | 須額外管理 iDRAC / BIOS / GPU FW |
| 監控指標來源 | vSphere + node-exporter | iDRAC + node-exporter + DCGM |
| 備份策略 | 可 vSphere snapshot（**勿用於 etcd VM**）+ rke2 snapshot | 僅 rke2 snapshot + 本地 OS backup |

> **特別警示**：**不可使用 vSphere VM snapshot 作為 etcd server 的備份**。VM snapshot 是 crash-consistent，可能凍結 etcd 在不一致狀態，還原後無法保證 etcd quorum。請只用 RKE2 內建的 `etcd-snapshot save`（[06](06-RKE2-Backup-and-Restore.md)）。

---

## 4. 升級順序的特別考量

1. **永遠先升 Server（VM）→ 再升 CPU Worker（VM）→ 最後升 GPU Worker（HGX）**
2. 升 GPU Worker 前需與 Run:ai 團隊協調：
   - 暫停新的 training job
   - 等待 running job 結束或主動 checkpoint
   - drain 後 RKE2 升級
   - GPU Operator 自動重建 nvidia 相關 pods（約 10 分鐘）
   - uncordon 後恢復排程
3. 跨 minor 升級時，**先在 VM CPU Worker 驗證 1 個節點**，再升 GPU 節點

---

## 5. 故障場景應對

### 5.1 單一 VM Server 故障

- HA 三節點容忍 1 個失敗
- 處理：vSphere 重建 VM → 重新 `INSTALL_RKE2` → 加入叢集 → 從 etcd 自動同步
- **不需要從 snapshot 還原**

### 5.2 兩個 VM Server 故障（quorum lost）

- etcd 失去 quorum，整個叢集失去寫入能力
- 處理：依 [06 §5.2](06-RKE2-Backup-and-Restore.md) 還原流程
- **預防**：DRS Affinity Rule 確保 3 個 Server VM 分散在不同 ESXi 主機

### 5.3 整個 vSphere 環境失效

- 3 Server 全失
- HGX 仍存活，但 control plane 全失 → 工作負載仍跑但無 API
- 處理：依 [06 §5.3](06-RKE2-Backup-and-Restore.md) 跨環境還原至備援 vSphere

### 5.4 HGX 實體機故障

- Run:ai 工作負載全失
- 處理：硬體送修期間，叢集本身正常運作；Run:ai 部分功能無 GPU 可用
- **規劃建議**：若 SLA 嚴格，建議至少 2 台 GPU 節點（**本案僅 1 台**為已知風險點，應於文件中對使用者揭露）

---

## 6. DRS / Affinity 規則建議（VMware 端）

| 對象 | 規則 | 目的 |
|------|------|------|
| 3 個 Server VM | **Anti-Affinity**：分散到不同 ESXi 主機 | 避免單一主機故障導致 quorum lost |
| 2 個 CPU Worker VM | **Anti-Affinity**：分散到不同 ESXi 主機 | 避免單主機故障導致 worker 全失 |
| Server VM | **VM-Host Affinity (should)**：固定在低延遲儲存 ESXi 主機群 | etcd 性能 |

---

## 7. BMC / iDRAC 整合（HGX）

| 項目 | 建議 |
|------|------|
| iDRAC 網段 | 與管理網隔離 |
| 韌體版本 | 與 Dell HGX 平台 release notes 對齊 |
| 監控 | Prometheus 透過 `redfish_exporter` 抓取 BMC 指標 |
| 遠端重啟流程 | 維運手冊應記載 iDRAC URL / 帳密保管位置 |
| 開機自動化 | 啟用 Power Restore Policy = Always On（避免電源中斷後需手動開機） |

---

## 8. NVIDIA Driver 版本對齊（HGX 專屬）

GPU Operator 部署 driver container 後，driver 版本與以下因素相關：

- GPU Operator chart 版本
- HGX H100/H200 GPU **最低要求 driver 版本**（如 H200 需 ≥ 545.x）
- CUDA 應用容器需求的 driver 版本

> **檢查相容性**：[NVIDIA Compatibility Matrix](https://docs.nvidia.com/datacenter/tesla/drivers/) 與 [GPU Operator Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html)。

---

## 9. 已知風險與緩解清單

| 風險 | 影響 | 緩解 |
|------|------|------|
| 單一 GPU 節點 | GPU 失效時 Run:ai 訓練全停 | 規劃增購第 2 台 GPU 節點 |
| 3 Server 同一 ESXi | 主機故障 = quorum lost | DRS Anti-Affinity |
| etcd VMDK 在共享儲存 | 高延遲影響整個叢集 | 改本地 SSD 或低延遲 tier |
| MTU 不一致 | Pod 間部分流量失敗 | 統一網段 MTU 規格 |
| Run:ai version skew | RKE2 升至 1.36 後 Run:ai 不支援 | 升級前查 Run:ai support matrix |
| 維運人員不熟 RKE2 | 故障處理時誤判 | 本文件庫 + 季度演練 |
| HGX BIOS/iDRAC 帳密遺失 | 無法遠端維運 | 密碼保管於組織密碼管理系統 |
| vSphere snapshot 誤用為備份 | etcd 還原失敗 | 文件明示 + 維運培訓 |

---

## 10. 維運交接清單

當新人接手本叢集時，應交付：

- [ ] 本文件庫（`doc-RKE2/`）
- [ ] 拓撲圖（含 VM / 實體機 / LB / 網段）
- [ ] IP / hostname 清單
- [ ] HAProxy / LB 設定檔
- [ ] RKE2 token 保管位置
- [ ] etcd 備份位置（本機 + S3）
- [ ] kubeconfig 取得方式
- [ ] iDRAC 與 vCenter 登入資訊
- [ ] 監控告警群組（Slack / email）
- [ ] On-call 升級 / 還原演練紀錄
- [ ] 與 Run:ai 團隊的窗口聯絡資訊
