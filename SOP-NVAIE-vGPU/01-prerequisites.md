# 01 — 先決條件與軟體取得

> 對應流程：階段 0（軟體授權與下載）+ 階段 1（ESXi/vCenter 環境就緒）
> 完成本章節後，你應該已經有：可用的 ESXi 8.0U3 + vCenter 8.0U3、NGC 帳號、下載好的 VIB / Guest Driver / DLS 映像、有效的 NVAIE 訂閱授權與 H200 NVL 已實體裝機。

---

## 1. 硬體先決條件

| 項目 | 要求 |
|------|------|
| 伺服器認證 | **NVIDIA-Certified System** 且**符合 ESXi 8.0U3 P06+ build** 在伺服器廠商 HCL 上 |
| GPU | **NVIDIA H200 NVL × 2**（PCIe 141 GB） |
| GPU 安裝 | 兩張卡插在伺服器支援的 PCIe 插槽，BIOS 已啟用 GPU 槽位 |
| 電源 | H200 NVL 為 600 W TDP，請確認電源與線材足夠 |
| 冷卻 | H200 NVL 為被動散熱，伺服器內部風流必須符合廠商規範 |
| **VMware 授權 edition** | **vSphere Foundation** 或 **vSphere Enterprise Plus**<br>（NVIDIA vGPU on ESXi 需要 Enterprise Plus 或 vSphere Foundation；Standard / Essentials 不可用。來源：[NVIDIA Virtual GPU Software for VMware vSphere Release Notes](https://docs.nvidia.com/vgpu/20.0/grid-vgpu-release-notes-vmware-vsphere/index.html)；NVAIE 部署相容性以 [NVAIE 8.1 Support Matrix](https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html) 為準） |

> 認證系統查詢：https://www.nvidia.com/en-us/data-center/products/certified-systems/

---

## 2. BIOS 設定（伺服器層級）

進入伺服器 BIOS / iLO / iDRAC，依下表確認：

| 設定項目 | 設定值 | 用途 |
|----------|--------|------|
| Hyperthreading | 啟用 | 提供足夠 vCPU 給 vGPU VM |
| Power Setting | High Performance | 避免 CPU throttling 影響 GPU 工作 |
| CPU Performance | Enterprise / High Throughput | |
| VT-d / IOMMU（Intel）/ AMD-Vi（AMD） | 啟用 | vGPU 與 DirectPath I/O 均需要 |
| Memory Mapped I/O above 4GB | 啟用 | H200 NVL 需大量 64-bit MMIO 空間 |
| Above 4G Decoding | 啟用（若 BIOS 有獨立選項） | |
| SR-IOV | 啟用（若 BIOS 有獨立選項） | 部分平台需要 |
| Resizable BAR | 啟用 | 改善大 BAR GPU 初始化 |
| Secure Boot | 視情況（見下方說明） | 若啟用，須使用簽署的 Driver |

> Secure Boot 若啟用：vGPU Guest Driver `.run` 安裝會需要 MOK 簽署流程，本手冊預設**未啟用 Secure Boot 以簡化流程**；如必須啟用，請另行查閱 [NVIDIA Driver Signing Guide](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html#runfile)。

---

## 3. 軟體版本相容性確認

本手冊以 **NVAIE Infra 8.1** 為主要範例。

| 元件 | 版本 |
|------|------|
| VMware ESXi | **8.0 Update 3 P06 或更新的 8.0 update**（build 對應 vCenter 8.0U3）<br>⚠️ NVIDIA vGPU 20.0 / 20.1 release notes 明確要求 **ESXi 8.0u3 P06 and later updates**；早於 P06 的 8.0U3 build 不在支援矩陣內，可能造成 VIB 無法載入或 VM 開機失敗。<br>來源：[NVIDIA Virtual GPU Software for VMware vSphere Release Notes](https://docs.nvidia.com/vgpu/20.0/grid-vgpu-release-notes-vmware-vsphere/index.html) |
| VMware vCenter Server | 8.0 Update 3（VCSA），對應 ESXi 8.0U3 P06+ build |
| NVAIE Host Software（VIB） | NVAIE Infra 8.1 對應的 `NVIDIA-AIE_ESXi_*.vib`（R595 分支，實際檔名以 NGC 下載為準） |
| vGPU Software | 20.1 |
| Data Center / Grid Driver | 595.71.05（R595） |
| Guest OS | Ubuntu Server **24.04 LTS**（amd64） |
| DLS Virtual Appliance | 3.6.x（部署前請至 [DLS Release Notes](https://docs.nvidia.com/license-system/latest/nvidia-license-system-release-notes/index.html) 確認最新版） |

> vSphere 8.0U3 屬「`8.0 and later`」官方支援字串範圍，落在 8.1 與 7.5 LTSB 兩條 branch 的支援矩陣內。執行前請以最新 Support Matrix 為準：
>
> - [Support Matrix 8.1](https://docs.nvidia.com/ai-enterprise/release-8/latest/support/support-matrix.html)
> - [Support Matrix 7.5 LTSB](https://docs.nvidia.com/ai-enterprise/release-7/7.5/support/support-matrix.html)

### 3.1 改採 NVAIE 7.5 LTSB 的調整

若選擇 7.5 LTSB（長期支援 3 年），把上表替換為：

| 元件 | 7.5 LTSB 版本 |
|------|--------------|
| VIB | R580 分支 `NVIDIA-AIE_ESXi_*.vib` |
| vGPU Software | 19.5 |
| Driver | 580.159.03 |

其餘步驟（VIB 安裝、DLS、VM 建立、Guest 設定）流程不變，**僅檔名與版本號替換**。

---

## 4. NGC / Licensing Portal 帳號

| 項目 | 用途 |
|------|------|
| **NVIDIA Enterprise 帳號**（綁定 NVAIE 訂閱授權） | 取得授權、產生 DLS Token |
| [NGC 個人 / 組織帳號](https://ngc.nvidia.com) | 下載 VIB、vGPU Guest Driver、DLS 映像、容器映像 |
| **NGC API Key** | CLI / Docker `nvcr.io` 登入用，由 NGC 右上角 Setup → Generate API Key |

> H100 / H200 NVL 購買時通常已含 5 年訂閱授權，請聯絡採購窗口確認授權已掛上你的 NVIDIA Enterprise Org，否則 Licensing Portal 看不到 Entitlement。

---

## 5. 軟體下載清單（建議事先放入 ESXi Datastore 或管理機）

放在管理工作站上（之後會分別上傳到 ESXi Datastore / 部署到 VM）：

| 檔案 | 來源 | 用途 |
|------|------|------|
| `VMware-ESXi-8.0U3-*.iso` | Broadcom Customer Portal | 安裝 ESXi 8.0U3（若主機尚未安裝） |
| `VMware-VCSA-all-8.0U3-*.iso` | Broadcom Customer Portal | 部署 vCenter 8.0U3 |
| `NVIDIA-AIE_ESXi_<8.1 對應版本>.vib` | NGC：NVAIE Infrastructure → Host Driver | 安裝在 ESXi（章節 02） |
| **vGPU Guest Driver ZIP**（例：`NVIDIA-GRID-Linux-KVM-595.<patch>-vGPU20.<n>.zip`），解壓後取出：<br>‣ `nvidia-linux-grid-595_595.<patch>_amd64.deb`（Ubuntu deb，**本手冊主用安裝法**）<br>‣ `NVIDIA-Linux-x86_64-595.<patch>-grid.run`（備用，章節 07 排查時使用） | NGC：NVAIE Infrastructure → Linux Guest Driver | Ubuntu 24.04 VM 安裝 vGPU Grid Driver（章節 05） |
| DLS Virtual Appliance OVA | NGC：NVIDIA License System → DLS | 部署 License Server VM（章節 03） |
| `ubuntu-24.04.<patch>-live-server-amd64.iso` | https://releases.ubuntu.com/24.04/ | Guest OS 安裝媒體 |
| Client Configuration Token `.tok`（之後在 DLS 介面產生） | DLS Web UI | Guest 套用授權（章節 03 → 05） |
| **NGC CLI**：`ngccli_linux.zip`（amd64 Linux） | https://org.ngc.nvidia.com/setup/installers/cli | 在 VM 內登入 NGC、後續下載容器與工件（章節 05） |

> **NGC CLI 範例下載指令**（在管理機執行，需先 `ngc config set` 輸入 API Key）：
>
> ```bash
> # vGPU Guest Driver（資源路徑：nvidia/vgpu/vgpu-guest-driver-<N>:<version>）
> # NVAIE 8.1 對應 vGPU 20.x，例如：
> ngc registry resource download-version "nvidia/vgpu/vgpu-guest-driver-20:<version>"
> ```
> 實際版本字串請至 [NGC Catalog](https://catalog.ngc.nvidia.com/) 搜尋 `vgpu-guest-driver-20` 取得；NVAIE 7.5 對應 `vgpu-guest-driver-19`。

---

## 6. 網路規劃

| 網段 | 用途 | 注意事項 |
|------|------|---------|
| Management 網段 | ESXi Host / vCenter Web UI / SSH | 建議獨立 VLAN |
| VM Data 網段 | 2 個測試 VM + DLS VM | 必須能互通 |
| **DLS ↔ vGPU VM** | TCP 443 / 80 | vGPU VM 開機後向 DLS 租用授權；防火牆需放行 |
| 對外網段（選用） | NGC / Container Registry 拉取 | Guest 安裝後需要對外下載 driver / Docker image |

> DLS 部署於地端時，兩個 vGPU VM 必須能 Layer 3 連到 DLS VM 的 TCP 443（HTTPS）。預設安裝會同時開啟 80，但建議走 HTTPS。

---

## 7. ESXi 8.0U3 + vCenter 8.0U3 安裝重點（簡述）

本手冊假設讀者已具備 VMware 基礎觀念，下列為串接 NVAIE 流程的最小確認項：

### 7.1 ESXi 8.0U3 主機

- 安裝完成後登入 DCUI，設定 Management Network IP / DNS / Hostname。
- 透過 vSphere Host Client（`https://<esxi-ip>/ui`）登入 → **Manage → Services**，啟動 `TSM`（ESXi Shell）與 `TSM-SSH`，並依需求設定為 *Start automatically*（建置完成後可關閉以提升安全性）。
- 確認在 BIOS / iLO 已可看到兩張 H200 NVL；ESXi 上可透過 `lspci | grep -i nvidia` 驗證（章節 02 會做完整驗證）。

### 7.2 vCenter Server 8.0U3（VCSA）

- 從管理機掛載 VCSA ISO，執行 `vcsa-ui-installer\<os>\installer` 進行兩階段部署。
- 部署目標：你的 ESXi 8.0U3 主機（單機環境）。
- 建立 SSO Domain（預設 `vsphere.local`）、設定 administrator 密碼。
- 完成後登入 `https://<vcenter-ip>/ui`，建立：
  - **Datacenter**（例：`DC-LAB`）
  - **Cluster**（例：`CL-NVAIE`）；單機環境 DRS / HA 可先關閉
  - 將 ESXi 8.0U3 主機 **Add Host** 加入 Cluster

> 從這一刻起，所有 vGPU 相關設定都建議透過 **vCenter Web Client** 完成，而非 ESXi Host Client，原因是 vGPU profile 選單在 vCenter 才會完整列出。

---

## 8. 完成檢查清單（章節 01 結束時應達成）

- [ ] 伺服器 BIOS 已依 §2 設定（VT-d / Above 4G / MMIO above 4GB 等都啟用）
- [ ] 兩張 H200 NVL 已實體裝機，BIOS POST 可見
- [ ] **ESXi 8.0U3 P06 或更新的 8.0 update** 已安裝（依 NVIDIA vGPU 20.0/20.1 VMware vSphere Release Notes），管理網路通；SSH 已啟用
- [ ] vCenter 8.0U3（VCSA）已部署，對應 ESXi 8.0U3 P06+ build；ESXi 已 Add Host 進 Cluster
- [ ] VMware 授權 edition 為 vSphere Foundation 或 Enterprise Plus
- [ ] NGC 帳號可登入、API Key 已產生
- [ ] NVIDIA Licensing Portal 看得到 NVAIE Entitlement（H200 NVL 訂閱）
- [ ] 已下載 `NVIDIA-AIE_ESXi_*.vib`（NVAIE 8.1）、**vGPU Guest Driver ZIP**（取出 `nvidia-linux-grid-*.deb`）、DLS OVA、Ubuntu 24.04 ISO、`ngccli_linux.zip`
- [ ] 管理機可連線 ESXi（22/443）與 vCenter（443）

→ 全部 ✅ 後，前往 [02-esxi-host-vib.md](02-esxi-host-vib.md)。
