# 安裝後設定

> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/quick-starts/infra-admin-quick-start  
> 來源：https://run-ai-docs.nvidia.com/self-hosted/getting-started/quick-starts/platform-admin-quick-start  
> 整理日期：2026-05-20  
> 目標版本：NVIDIA Run:ai v2.25

---

## 1. 概述

Run:ai Cluster 連線成功後，需完成以下設定使系統可供使用者正常使用：

| 設定項目 | 重要性 | 說明 |
|----------|--------|------|
| 修改初始密碼 | **必做** | 首次登入後立即修改 |
| SSO 設定 | 建議 | 整合企業 LDAP/AD |
| Email 伺服器 | 建議 | 用戶通知功能 |
| 建立 Department/Project | **必做** | 資源配額管理基礎 |
| 建立用戶帳號 | **必做** | 研究員存取權限 |
| Node Pool 設定 | 建議 | GPU 節點分組管理 |

---

## 2. 初始密碼修改

首次登入後立即執行：

1. 登入 Run:ai UI（`https://runai.<DOMAIN>`）
2. 右上角點擊帳號圖示 → **Change Password**
3. 設定強密碼（需含大小寫、數字、特殊字元）

---

## 3. SSO 設定（選用但建議）

Run:ai 支援透過 **SAML 2.0** 或 **OpenID Connect (OIDC)** 整合企業 Identity Provider：

- **Active Directory / LDAP**：需透過支援 SAML 或 OIDC 的 IdP（例如 Keycloak、Okta）
- **設定路徑**：Run:ai UI → Settings → SSO

> 詳細設定步驟請參閱 [Run:ai SSO 官方文件](https://run-ai-docs.nvidia.com/self-hosted/infrastructure-setup/advanced-setup/control-plane-config)

---

## 4. Email 伺服器設定（選用但建議）

設定後，系統可傳送用戶邀請信、密碼重設通知：

1. Run:ai UI → Settings → Email Server
2. 填入 SMTP 伺服器設定：
   - SMTP Host / Port
   - 認證帳號密碼
   - 寄件者 Email 地址

---

## 5. 建立 Department 與 Project

Run:ai 使用兩層資源配額結構：

```
Cluster
  └── Department（部門）
        └── Project（專案）
              └── 工作負載（Workload）
```

### 5.1 建立 Department

1. Run:ai UI → Resources → **Departments**
2. 點擊「New Department」
3. 設定：
   - **Name**：部門名稱
   - **GPU Quota**：最大 GPU 配額

### 5.2 建立 Project

1. Run:ai UI → Resources → **Projects**
2. 點擊「New Project」
3. 設定：
   - **Name**：專案名稱
   - **Department**：歸屬部門
   - **GPU Quota**：GPU 配額（可使用整數或分數，例如 `1.5`）
   - **Node Pool**：指定可使用的節點池

---

## 6. 建立用戶帳號

### 6.1 透過 Run:ai UI 建立本地帳號

1. Run:ai UI → Settings → **Users**
2. 點擊「Invite User」
3. 填入 Email，選擇角色：

| 角色 | 說明 |
|------|------|
| Administrator | 系統管理員，完整權限 |
| Research Manager | 可管理 Project 與工作負載 |
| Researcher | 可提交工作負載 |
| Viewer | 唯讀，只能查看狀態 |

### 6.2 分配 Project 存取權限

1. 進入對應 Project → **Access Control**
2. 新增用戶並指定角色

---

## 7. Node Pool 設定（建議）

Node Pool 可將 GPU 節點分組，提供不同規格的資源配置給不同專案：

```bash
# 確認 GPU 節點標籤
kubectl get node <xe9780-node-name> --show-labels | grep nvidia

# 為節點新增自訂標籤（用於 Node Pool 識別）
kubectl label node <xe9780-node-name> runai-pool=gpu-pool-01
```

1. Run:ai UI → Resources → **Node Pools**
2. 建立 Node Pool，設定節點選擇條件（Node Selector）

---

## 8. runai CLI 安裝（研究員工具）

研究員可透過 CLI 提交工作負載：

```bash
# 解壓並安裝（以 Linux amd64 為例）
tar -xzf runai-linux-amd64.tar.gz
sudo mv runai /usr/local/bin/

# 登入
runai login --url https://runai.<DOMAIN>

# 確認連線
runai cluster list
```

> **待驗證**：runai CLI 下載方式請從 Run:ai UI → Clusters 頁面取得官方下載連結（官方文件未提供固定下載 URL，需從 UI 取得）。連結格式因版本而異，不可直接手動填寫。

---

## 9. 驗證整體功能

提交測試工作負載確認系統正常運作：

```bash
# 設定預設 Project
runai config project <project-name>

# 提交測試工作負載（要求 1 GPU）
runai submit test-job \
  --image nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04 \
  --gpu 1 \
  -- /bin/sh -c "echo GPU test OK"

# 查看工作負載狀態
runai list jobs

# 查看 Logs
runai logs test-job

# 刪除測試工作負載
runai delete test-job
```

---

## 10. 下一步

→ [06-Validation-and-Monitoring.md](06-Validation-and-Monitoring.md) — 系統持續監控與健康確認
