# 03 — 部署 DLS License Server 與產生 Client Token

> 對應流程：階段 3
> 完成本章節後，你會有一台運行中的 DLS Virtual Appliance VM、已從 NVIDIA Licensing Portal 匯入 NVAIE 授權至本機 DLS、產生並下載好 **Client Configuration Token（`.tok`）** 供章節 05 套用至兩個 vGPU VM。

---

## 1. 為什麼選 DLS（而非 CLS）

| 比較 | DLS（地端） | CLS（雲端，NVIDIA 託管） |
|------|------------|------------------------|
| 部署 | 客戶自建 VM | 不需部署 |
| 連網 | 隔離環境可 | 需外網連 NVIDIA Licensing Cloud |
| 適用 | 內網部署、資安要求高、本手冊預設 | 簡單環境、有外網 |

> 若你的 vGPU VM 可順暢連到 NVIDIA Licensing Cloud，且不介意走外網，**CLS 流程更簡單**（跳過本章節 §4–§6，直接在 Licensing Portal 產生 CLS Token 後到章節 05 套用即可）。本手冊以 DLS 為主例。

---

## 2. DLS VM 規格與網路

| 項目 | 最低需求 | 本手冊建議 |
|------|---------|----------|
| vCPU | 4 | 4 |
| RAM | 8 GB | 8 GB |
| Disk | 15 GB | 30 GB（thin） |
| 網路 | 1 × VMXNET3，靜態 IP | 與 vGPU VM 同網段、能互通即可；DLS VM **不需要直接連 Internet** |
| 連接埠（DLS 對外） | TCP 443（HTTPS）/ TCP 80（HTTP，備用） | 確保 vGPU VM → DLS 防火牆放行 |

> **DLS 連網需求釐清（更正：DLS VM 本身不需要直接 Internet）：**
> - **管理工作站／瀏覽器**：需能登入 [NVIDIA Licensing Portal](https://licensing.nvidia.com)，在那邊建立 Service Instance / License Server、下載 **License Server 綁定檔（`.bin`）** 並 **手動上傳**到 DLS Web UI。
> - **DLS VM**：只需與 vGPU VM 互通（TCP 443/80）；綁定過程透過管理者手動匯入 `.bin` 即可完成，DLS VM 不必能 Layer 3 連到 Licensing Portal。
> - 來源：[NVIDIA License System User Guide](https://docs.nvidia.com/license-system/latest/nvidia-license-system-user-guide/index.html)。
>
> 隔離（air-gapped）環境部署時，這個分工尤其重要：只要管理機可暫時連外完成下載 `.bin`、再以隨身裝置或內部檔案傳輸把 `.bin` 送進 DLS Web UI，DLS VM 全程在隔離網路內仍可運作。

---

## 3. 取得 DLS Virtual Appliance OVA

1. 登入 [NVIDIA Licensing Portal](https://licensing.nvidia.com)。
2. 左側 **SOFTWARE DOWNLOADS** → **License System** → 選 **DLS Virtual Appliance** → **vSphere** 平台 → 下載對應的 `nls-<version>-bios.zip` 或 `.ova`（依介面提供格式）。

> 截至 2026-05，DLS 穩定版約 3.6.x；版本號頻繁更新，以官網最新版為準（[DLS Release Notes](https://docs.nvidia.com/license-system/latest/nvidia-license-system-release-notes/index.html)）。

---

## 4. 部署 DLS VM（OVA 匯入 vCenter）

1. 在 vCenter Web Client：右鍵 Cluster / Host → **Deploy OVF Template**。
2. **Local file** → 選擇下載的 `.ova`（或解壓的 `.ovf` + `.vmdk`）。
3. VM 名稱：`dls-licsrv-01`（自定）。
4. 選 Datacenter / Cluster / Datastore。
5. 網路：選與 vGPU VM 同一網段的 Port Group。
6. **Customize template**：
   - 設定**靜態 IP**（例：`10.10.10.50/24`）、Gateway、DNS。
   - DLS 預設管理員密碼（系統初次設定時會再要求變更）。
7. Finish，等待匯入完成（≈ 5–10 分鐘，依儲存效能）。
8. 開機 DLS VM。

---

## 5. 初始設定 DLS

### 5.1 開啟管理介面

開機完成後，瀏覽器訪問：

```
https://<dls-vm-ip>
```

> 第一次連線會出現自簽憑證警告，接受即可（生產環境請替換憑證）。

### 5.2 完成 OS 層管理員註冊

DLS 內部有兩層帳號：

| 帳號 | 用途 |
|------|------|
| `dls_admin` | OS 層級管理員（DLS 系統管理） |
| `rsu_admin` | RSU（Remote System Update）管理員 |
| Application Users | 應用層 RBAC（管理 Service Instance / License Pool） |

依首頁精靈步驟設定 `dls_admin` 密碼 → 登入後再建立第一個 Application User（例如 `nvaie-admin`）。

---

## 6. 從 NVIDIA Licensing Portal 註冊 DLS Instance

DLS 需要先在 Licensing Portal 上**註冊（bind）**才能領取授權。

### 6.1 在 DLS Web UI 取得 DLS Instance ID

登入 DLS Web UI（使用 Application User）→ **SERVICE INSTANCES** → 紀錄當前 DLS 的 *Instance ID*（UUID 格式）。

### 6.2 在 Licensing Portal 註冊

1. 登入 [Licensing Portal](https://licensing.nvidia.com)。
2. **SERVICE INSTANCES** → **CREATE SERVICE INSTANCE** → 選 **On-Premises（DLS）**。
3. 輸入 DLS Instance ID、Name（例：`dls-licsrv-01`）→ Create。
4. 在 **LICENSE SERVERS** → **CREATE LICENSE SERVER**：
   - 名稱：`nvaie-h200nvl-pool`
   - 選擇要綁定的 Service Instance（剛建立的 DLS）
   - **ADD FEATURES**：選擇 **NVIDIA AI Enterprise**（你採購的 features），輸入欲分配的 license 數量（≥ 2，因為兩個 VM 各租用一個）
5. Save → 系統會產生一份 **License Server 綁定檔（`.bin`）**，下載後上傳至 DLS。

### 6.3 將綁定檔匯入 DLS

回到 DLS Web UI → **SERVICE INSTANCES** → 上傳剛下載的 `.bin` → 完成綁定。

完成後，DLS Web UI 的 **DASHBOARD** 應顯示：
- License Server: `nvaie-h200nvl-pool`
- Available Features: `NVIDIA AI Enterprise`（如：`2 / 2` 可用）

---

## 7. 產生 Client Configuration Token（`.tok`）

每個 vGPU Guest VM 都會用同一份 Token 向 DLS 租用授權。

### 7.1 在 DLS Web UI 產生

1. **SERVICE INSTANCES** → 選你的 instance → **ACTIONS** → **Download Client Configuration Token**。
2. 選擇授權對象（License Server `nvaie-h200nvl-pool`）。
3. 設定有效期限（建議 1–2 年）。
4. **Server address preferences**：選擇 Token 內寫入的 DLS 連線位址形式：
   - **FQDN（預設）**：Client 端必須有可解析 DLS hostname 的 DNS（含正向解析；若你的環境 PTR 不完整時務必確認 A record 正常）。
   - **IPv4 address**：適合**沒有可靠 DNS 或 air-gapped 環境**。本手冊在隔離 / 簡化測試環境**建議選 IPv4**，避免 client 解析失敗。
   - **IPv6 address**：依環境是否走 IPv6。
   - 來源：[NVIDIA License System Quick Start Guide](https://docs.nvidia.com/license-system/latest/nvidia-license-system-quick-start-guide/index.html)
5. 下載產生的 `client_configuration_token_<timestamp>.tok` 至管理機。

> 同一份 Token 可給多個 VM 使用；本範例兩個 VM 共用此 Token。
>
> **驗證 DNS（若選 FQDN）：** 在 vGPU VM 內 `getent hosts <DLS-FQDN>` 應該回傳 DLS IP；若回傳空 → 改用 IP 形式重新產生 Token，或先把 DNS / `/etc/hosts` 補齊。

### 7.2 將 Token 保留備用

將 `.tok` 暫存於管理機，章節 05 §6 會 SCP 或透過 VMware Tools 複製到兩個 vGPU VM 內。

---

## 8. 完成檢查清單

- [ ] DLS VM 已部署、開機、可透過 `https://<dls-ip>` 登入管理介面
- [ ] Licensing Portal 已建立 Service Instance（綁定本 DLS）
- [ ] Licensing Portal 已建立 License Server `nvaie-h200nvl-pool` 並分配至少 2 個 NVAIE feature
- [ ] DLS Dashboard 顯示 Available 數量正確（例如 `2 / 2`）
- [ ] 已下載 `client_configuration_token_*.tok`，**檔名請妥善保存**
- [ ] 從 ESXi 與未來的 vGPU VM 網段都能 `curl -k https://<dls-ip>` 取得 DLS 頁面（網路通）

→ 全部 ✅ 後，前往 [04-vm-creation-and-vgpu-profile.md](04-vm-creation-and-vgpu-profile.md)。

---

## 9. 參考連結

- DLS 官方 User Guide：https://docs.nvidia.com/license-system/latest/nvidia-license-system-user-guide/index.html
- NVAIE Licensing Guide：https://docs.nvidia.com/ai-enterprise/planning-resource/licensing-guide/latest/licensing.html
- DLS Release Notes：https://docs.nvidia.com/license-system/latest/nvidia-license-system-release-notes/index.html
