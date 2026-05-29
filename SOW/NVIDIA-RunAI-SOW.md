# NVIDIA Run:ai 建置時程評估與風險清單

> 評估日期：2026-05-20
> 目標版本：NVIDIA Run:ai v2.25
> 建置情境：Dell XE9780（GPU 節點）× 1 + Dell R670（管理節點）× 2，Bare Metal K8s + Self-hosted 模式
> **方案框架：參考 NVIDIA Enterprise Reference Architecture 2-8-9-400 HGX 設計模式（實際 Dell AI Factory endorsement 需依 Dell / NVIDIA 確認）**
> 評估角度：初學者（具備基本 IT 概念、但首次接觸 K8s / NVIDIA GPU Operator / Run:ai 者）

---

## 整體建置時程評估（初學者）

文件本身的部署清單預估有經驗人員需 **6–11 小時**。對於初學者，評估如下：

| 階段 | 工作內容 | 初學者預估天數 |
|------|----------|----------------|
| **前置學習** | K8s 概念、Helm、kubectl 基本操作、文件閱讀 | 3–5 天 |
| **硬體與 OS 準備** | Ubuntu 22.04 × 3 安裝、網路/IP 設定、NTP/Swap/核心參數 | 1–2 天 |
| **K8s 平台建置** | RKE2 或 kubeadm 安裝、雙 Master HA 設定、CNI 網路 ¹ | 2–4 天 |
| **K8s 基礎元件** | MetalLB、Ingress Controller、Storage Class、Prometheus | 1–2 天 |
| **GPU Operator** | 安裝、版本確認、CUDA 測試驗證 | 1–2 天 |
| **DNS 與 TLS 憑證** | FQDN 設定、萬用字元憑證申請（需跨部門協調）| 2–5 天 |
| **Run:ai 安裝** | Control Plane + Cluster（含 NGC API Key 申請）| 1–2 天 |
| **安裝後設定與驗證** | 帳號、Department/Project、CLI 測試 | 1 天 |
| **問題排解緩衝** | 環境差異、設定錯誤、版本衝突 | 3–5 天 |
| **合計** | | **15–28 天** |

> DNS/TLS 申請與 NVIDIA 授權採購若需走採購流程，可能另外增加 5–10 個工作天（不含在上表）。  
> ¹ **Kubernetes 發行版說明**：本時程以 RKE2 / kubeadm 為基準估算，兩者均列於 Run:ai 官方支援矩陣。Dell 官方 2-8-9-400 brief（XE9680 參考平台）所列的 orchestration 路徑為 Upstream Kubernetes 或 Red Hat OpenShift Container Platform；若需完整對齊 Dell AI Factory endorsed stack，請由 Dell / NVIDIA SE 確認發行版選型，時程可能因此調整。

---

## 可能風險清單

### 技術相容性風險

1. **K8s 版本不符** — Run:ai v2.25 僅支援 K8s 1.33–1.35，初學者易從預設 apt repo 安裝到舊版本，導致安裝失敗。

2. **GPU Operator 版本混淆** — 官方文件**自身存在不一致**（支援矩陣表格寫 25.10–26.3，同頁另一段落仍有 25.3–25.10 的舊文字）。初學者容易跟錯段落。

3. **GPU 型號相容性未確認** — Dell XE9780 使用的 GPU 型號**待現場確認**，若 GPU 不在支援清單則無法繼續。

4. **Prometheus 版本過舊** — Run:ai 要求 kube-prometheus-stack ≥ 76.0，舊版本造成指標無法收集，症狀不明顯難以排查。

---

### 網路與憑證風險

5. **萬用字元 TLS 憑證** — v2.25 **預設啟用 Host-based Routing**，必須準備 `*.<DOMAIN>` 萬用字元憑證。自簽憑證會導致瀏覽器警告與 CLI 認證失敗；向 CA 申請需時。

6. **FQDN 強制要求** — Run:ai 不允許使用純 IP，所有節點必須能解析 FQDN。若組織沒有內部 DNS 伺服器，需另行架設或向 IT 申請。

7. **MetalLB IP 位址池規劃錯誤** — 若設定的 IP 範圍與現有網路設備衝突，LoadBalancer Service 將取得衝突 IP，導致服務不可達。

8. **IPv6 環境陷阱** — `nvcr.io` 僅有 IPv4 DNS 記錄；純 IPv6 環境需額外配置 NAT64/DNS64，對初學者是高難度課題。

---

### 安裝流程風險

9. **HA Control Plane 設定複雜** — 雙 Master 節點需要 VIP 或 Load Balancer（HAProxy + Keepalived / kube-vip），kubeadm 的 `--control-plane-endpoint` 必須在 `kubeadm init` 前確定，事後難以修改。

10. **Ingress Class 名稱不符** — Helm 安裝 Run:ai 時必須以 `--set global.ingress.ingressClass=<正確名稱>` 指定，拼錯或與叢集實際 IngressClass 不符，Ingress 資源不會被處理。

11. **Container Runtime cgroup 設定遺漏** — `containerd` 的 `SystemdCgroup` 預設為 `false`，必須手動改為 `true`。遺漏此設定 K8s 叢集表面正常，但 GPU 工作負載執行時出現不明顯錯誤。

12. **Control Plane 初始化等待誤判** — Control Plane 最多需 15 分鐘完成初始化，初學者可能誤判為卡住而重複嘗試，造成資源衝突。

---

### 授權與帳號風險

13. **NVIDIA NGC API Key 與 NVAIE 授權** — 需要 NGC 帳號才能拉取映像與 Helm Charts；Run:ai v2.20 後整合於 NVIDIA AI Enterprise 授權，採購流程可能耗時數週。

14. **離線（Air-gapped）環境需額外準備** — 若環境無法連線 `nvcr.io`、`gcr.io`、`quay.io`，需提前下載 **20 GB 以上**映像至內部 Registry，這是與連網環境完全不同的部署路徑。

---

### 維運風險

15. **核心版本自動更新破壞 GPU Operator** — Ubuntu 預設開啟自動更新，核心版本升級可能與已安裝的 NVIDIA Driver 不相容，導致 GPU Operator Pod 重啟失敗。部署時需手動鎖定核心版本。

16. **Swap 未徹底關閉** — `swapoff -a` 僅暫時關閉；若未修改 `/etc/fstab`，重開機後 Swap 恢復，K8s kubelet 啟動失敗。

17. **未來升級路徑受限** — 跨版本升級必須逐版進行（如 2.23→2.24→2.25），不可跳版，且必須先升 Control Plane 再升 Cluster；若直接跳版會導致資料損毀。

---

## 最高優先處理項目（建議部署前確認）

1. **Dell AI Factory / NVIDIA ERA 對齊確認**：與 Dell / NVIDIA SE 確認 Dell PowerEdge XE9780 是否屬於 Dell AI Factory 正式 endorsed configuration（NVIDIA ERA White Paper 說明的是通用 HGX 2-8-9-400 設計模式；Dell 官方 2-8-9-400 brief 及 NVIDIA ERA index 中明確列出的 Dell 參考平台為 XE9680，非本案 XE9780）
2. **XE9780 GPU 型號確認**：依採購 BOM / Dell Service Tag 確認 GPU 型號；依 Canonical Ubuntu 認證頁面（Cert ID: 202510-37982），XE9780 認證的 GPU 為 **B200 SXM6** 或 **B300 NVL8**，確認後對照 GPU Operator 26.3.x 支援清單
3. 組織是否有 DNS 伺服器可新增 A Record 與萬用字元記錄
4. TLS 萬用字元憑證的申請管道與時程
5. NVIDIA AI Enterprise 授權是否已取得或正在採購流程中（H200 購買可能附贈 5 年 NVAIE 訂閱）
