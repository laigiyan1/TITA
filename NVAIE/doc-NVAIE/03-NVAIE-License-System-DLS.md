# NVAIE 授權系統（License System）設定指南

> 來源：
> - https://docs.nvidia.com/ai-enterprise/deployment/vmware/latest/nls.html  
> - https://docs.nvidia.com/license-system/latest/  
> - https://docs.nvidia.com/ai-enterprise/planning-resource/licensing-guide/latest/licensing.html  
> 整理日期：2026-05-20

---

## 1. 授權模型概覽

- **每 GPU 授權**：「A software license is required for every GPU installed on the server or workstation that will host any software that is included with NVIDIA AI Enterprise.」（NVIDIA 官方授權指南原文）
- **GPU Passthrough 模式**：不需要 vGPU License Server（NLS client leasing），但執行 NVAIE 軟體的 GPU 仍需持有有效的 NVAIE per-GPU 訂閱授權。
- **vGPU 模式**：除了 NVAIE 訂閱授權外，還需要部署 License Server（CLS 或 DLS），VM 開機時透過 NLS client 向 License Server 租用浮動授權。
- **H100/H200 NVL 購買時已含 5 年訂閱授權**

---

## 2. 授權服務類型

### 2.1 CLS（Cloud License Service）
- 由 **NVIDIA 雲端託管**
- VM 需能連接到 NVIDIA 授權伺服器（需外網）
- 無需自建伺服器基礎設施
- 適合：有外網連線的環境

### 2.2 DLS（Delegated License Service）
- 部署在**客戶自己的地端環境**
- VM 透過內部網路取得授權
- 適合：隔離網路、資安要求高的環境
- **您的 vGPU 環境建議使用 DLS**

---

## 3. 授權運作機制

```
[VM with vGPU Guest Driver]
        ↓ 網路連線（TCP 443/80）
[License Server (CLS or DLS)]
        ↓ 驗證
[NVIDIA Licensing Portal]（只有 DLS 初始設定時需要）
```

- VM 開機時向 License Server 請求授權
- License Server 從授權池中分配一個浮動授權（Floating License）
- VM 關機後授權歸還給池中
- **授權租約機制**：定期更新（心跳監控）

---

## 4. DLS 部署需求

### 4.1 硬體/VM 需求

| 資源 | 最低需求 |
|------|----------|
| vCPU | 4 |
| RAM | 8 GB |
| 磁碟 | 15 GB |

### 4.2 支援的 Hypervisor
- VMware vSphere ✅
- Hyper-V ✅
- Citrix ✅
- KVM ✅
- Ubuntu（Bare Metal）✅

### 4.3 容器化部署選項
- Docker ✅
- Kubernetes ✅
- Podman ✅
- OpenShift ✅

### 4.4 網路連接埠需求

| 連接埠 | 用途 |
|--------|------|
| 443 | HTTPS（授權用戶端連線） |
| 80 | HTTP（備用） |
| 叢集特定連接埠 | HA 模式下節點間通訊 |

---

## 5. DLS 安裝流程

### 5.1 基本安裝步驟

```
1. 從 NVIDIA Licensing Portal 下載 DLS VM 映像檔
2. 在 VMware 上部署 DLS VM（OVA/OVF 格式）
3. 設定 VM 靜態 IP 位址
4. 使用瀏覽器存取 DLS 管理介面（https://<DLS-IP>）
5. 完成系統管理員帳號註冊
6. 從 NVIDIA Licensing Portal 匯入授權
7. 建立授權服務實例（Service Instance）
8. 產生 Client Configuration Token
9. 在每個 vGPU VM 上套用 Token
```

詳細安裝說明：https://docs.nvidia.com/license-system/latest/nvidia-license-system-user-guide/index.html

### 5.2 管理介面帳號類型

| 帳號類型 | 說明 |
|----------|------|
| dls_admin | OS 層級系統管理員 |
| rsu_admin | OS 層級 RSU 管理員 |
| Application Users | 應用程式層 RBAC 角色 |

---

## 6. 效能與擴展性

| 指標 | 數值 |
|------|------|
| 最大客戶端數 | 84,000 |
| 授權發放速率 | 5–13 clients/second |

---

## 7. 高可用性（HA）設定

DLS 支援 Primary/Secondary 雙節點 HA：
- Virtual IP Management
- 自動 Failover 監控
- Heartbeat 檢查機制

---

## 8. Client Token 設定（在 vGPU VM 中）

在 vGPU Guest VM（Ubuntu）中配置授權：

```bash
# 將 Client Configuration Token 放到指定目錄
sudo mkdir -p /etc/nvidia/ClientConfigToken/
sudo cp client_configuration_token_*.tok /etc/nvidia/ClientConfigToken/

# 重啟 NVIDIA 授權服務
sudo systemctl restart nvidia-gridd

# 驗證授權狀態
nvidia-smi -q | grep -A 5 "vGPU Software Licensed"
```

---

## 9. DLS 版本資訊（2026-05，版本號可能已更新）

| 版本類型 | 版本（截至 2026-05-20） |
|----------|------|
| 目前版本 | 3.6.0 – 3.6.1 |
| 長期支援版本（LTS） | 3.1.0 – 3.1.9（EOL 2027/06） |

> **注意**：DLS 版本號更新頻繁，以上版本為整理當時資訊，**部署前請至官方頁面確認最新版本**：  
> https://docs.nvidia.com/license-system/latest/nvidia-license-system-release-notes/index.html

---

## 10. 參考連結

| 文件 | URL |
|------|-----|
| NVIDIA License System 主頁 | https://docs.nvidia.com/license-system/latest/ |
| DLS User Guide | https://docs.nvidia.com/license-system/latest/nvidia-license-system-user-guide/index.html |
| NVAIE Licensing Guide | https://docs.nvidia.com/ai-enterprise/planning-resource/licensing-guide/latest/licensing.html |
| NVIDIA Licensing Portal | https://licensing.nvidia.com |
