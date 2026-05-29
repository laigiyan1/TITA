# 00 – RKE2 概覽與架構

> 來源：
> - [https://docs.rke2.io/](https://docs.rke2.io/)
> - [https://docs.rke2.io/architecture](https://docs.rke2.io/architecture)
> - [https://documentation.suse.com/cloudnative/rke2/latest/en/architecture.html](https://documentation.suse.com/cloudnative/rke2/latest/en/architecture.html)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. 產品定位

RKE2（又稱 **RKE Government**）是 SUSE / Rancher 推出的下一代企業級 Kubernetes 發行版，由原 RKE（Rancher Kubernetes Engine v1）演進而來。產品設計目標：

- **以單一二進位檔（single binary）部署**：所有節點安裝同一個 `rke2` 執行檔，依角色（server/agent）切換行為。
- **與上游 Kubernetes 緊密對齊**：不像 RKE1 那樣將控制平面元件封裝為 Docker 容器，RKE2 將控制平面元件以 **static pod** 形式由 kubelet 管理。
- **聚焦安全與合規**：預設配置可以最少操作員介入即通過 CIS Kubernetes Benchmark，並提供 FIPS 140-2 模式。
- **不依賴 Docker**：內建 containerd 與 runc 作為容器執行時。

> **產品差異對照**：
> - **與 RKE1 比較**：RKE2 不依賴 Docker；以 static pod 取代 Docker 容器執行控制平面元件；更貼近上游 Kubernetes 行為。
> - **與 K3s 比較**：RKE2 結合 K3s 的部署簡易性，但保留完整上游 Kubernetes 一致性；K3s 為邊緣/輕量場景優化，RKE2 為資料中心生產級場景優化。
> - 詳見 [Rancher 官方比較頁](https://ranchermanager.docs.rancher.com/v2.9/how-to-guides/new-user-guides/launch-kubernetes-with-rancher/rke1-vs-rke2-differences)。

---

## 2. 整體架構

![RKE2 架構總覽](../pic-RKE2/rke2-architecture-overview.png)

> *圖片來源：[docs.rke2.io – Architecture](https://docs.rke2.io/architecture)*

### 2.1 Server 節點（Control-Plane + etcd）

Server 節點以 `rke2 server` 模式執行 `rke2` 二進位檔，內含以下元件：

| 元件 | 形式 | 角色 |
|------|------|------|
| `rke2` binary | systemd 主進程 | 啟動器、生命週期管理 |
| `containerd` | systemd 子進程 | 容器執行時，內含 runc |
| `kubelet` | 主機進程（由 rke2 拉起） | 管理 static pod 與 daemonset |
| `kube-apiserver` | **Static Pod** | 叢集 API 入口（埠 6443） |
| `etcd` | **Static Pod** | 分散式 KV 儲存 |
| `kube-controller-manager` | **Static Pod** | 內建控制器 |
| `kube-scheduler` | **Static Pod** | Pod 排程器 |
| `cloud-controller-manager` | **Static Pod** | 雲端整合（無雲端時為空殼） |
| `kube-proxy` | **Static Pod** | Service 流量轉發 |
| `helm-controller` | 主機進程 | 自動部署內建 Helm Chart |

### 2.2 Agent 節點（Worker）

Agent 節點以 `rke2 agent` 模式執行，元件較少：

| 元件 | 形式 | 角色 |
|------|------|------|
| `rke2` binary | systemd 主進程 | 啟動器 |
| `containerd` | systemd 子進程 | 容器執行時 |
| `kubelet` | 主機進程 | 節點代理 |
| `kube-proxy` | **Static Pod** | Service 流量轉發 |
| RKE2 Service Load Balancer | 內建於 rke2 binary | Agent → Server 連線負載分擔 |

> **Static Pod 機制**：RKE2 將控制平面元件清單寫入由 kubelet 監控的 static pod manifest 目錄（路徑由 RKE2 內部管理，一般無需直接操作）。**這意味著用 `kubectl delete pod` 刪除這些 Pod 不會真正關閉控制平面**，kubelet 會立即重建；必須透過停止 `rke2-server` systemd 服務才能停止。

---

## 3. 啟動順序

### 3.1 Server 節點啟動序列

來源：[docs.rke2.io – Architecture](https://docs.rke2.io/architecture)

1. **`rke2 server` 進程啟動**：從 `rancher/rke2-runtime` 映像解壓 binaries 與 manifests。
2. **containerd 啟動**。
3. **kubelet 啟動**。
4. **kubelet 監測 static pod 目錄**並依序啟動：
   - etcd → kube-apiserver → kube-controller-manager → kube-scheduler → cloud-controller-manager → kube-proxy
5. **helm-controller** 在 API server 就緒後啟動，依序套用內建 Helm Chart：
   - 網路 CNI（`rke2-canal` / `rke2-cilium` / `rke2-calico` / `rke2-flannel`，可選擇性疊加 `rke2-multus`）
   - `rke2-coredns`
   - `rke2-ingress-nginx` 或 `rke2-traefik`（v1.36+ 預設為 Traefik）
   - `rke2-metrics-server`
   - `rke2-runtimeclasses`、snapshot controller 等
6. **常駐執行**：直至接收終止訊號或 containerd 退出。

### 3.2 Agent 節點啟動序列

1. **`rke2 agent` 進程啟動**。
2. **containerd 啟動**。
3. **註冊至 Server**：透過 `/etc/rancher/rke2/config.yaml` 中的 `server:` 與 `token:` 向 Server 的埠 9345 註冊。
4. **kubelet 啟動**並接受 Server 派發工作負載。

---

## 4. 內建元件版本對照（stable vs latest channel）

來源：
- [docs.rke2.io v1.35.X Release Notes](https://docs.rke2.io/release-notes/v1.35.X)
- [docs.rke2.io v1.36.X Release Notes](https://docs.rke2.io/release-notes/v1.36.X)

| 元件 | v1.35.5+rke2r1（stable，**本文件主軸**） | v1.36.1+rke2r1（latest，2026-05-18 釋出） |
|------|------------------------------------------|---------------------------------------------|
| Kubernetes | v1.35.5 | v1.36.1 |
| containerd | v2.2.3-k3s1 | v2.2.3-k3s1 |
| etcd | v3.6.7-k3s1 | v3.6.7-k3s1 |
| CoreDNS | v1.14.3 | v1.14.3 |
| 預設 Ingress | **ingress-nginx** v1.14.5-hardened2 | **Traefik** v3.6.16（v1.36 起預設） |
| Cilium | v1.19.3 | v1.19.3 |
| Calico | v3.32.0 | v3.32.0 |
| Flannel | v0.28.4 | v0.28.4 |

> **重點差異**：v1.35 仍以 **ingress-nginx** 為預設 Ingress；v1.36 起預設改為 **Traefik**（ingress-nginx 將於 2026-03 EOL，v1.37 完全移除）。**升級至 v1.36 的既有叢集會保留原 ingress 不切換**。

---

## 5. 主要檔案路徑速查

| 路徑 | 用途 |
|------|------|
| `/etc/rancher/rke2/config.yaml` | 主要設定檔 |
| `/etc/rancher/rke2/config.yaml.d/*.yaml` | 子設定檔（依字母序合併） |
| `/etc/rancher/rke2/rke2.yaml` | 預設 kubeconfig（Server 節點） |
| `/var/lib/rancher/rke2/server/node-token` | HA token 來源 |
| `/var/lib/rancher/rke2/server/token` | 加密 etcd 機密資料的 token（**還原必備**） |
| `/var/lib/rancher/rke2/server/db/snapshots/` | etcd 快照預設位置 |
| `/var/lib/rancher/rke2/server/manifests/` | 自動部署 manifest 目錄 |
| `/var/lib/rancher/rke2/agent/etc/containerd/config.toml` | containerd 生成設定（**勿直接修改**） |
| `/var/lib/rancher/rke2/agent/etc/containerd/config.toml.tmpl` | containerd 自訂樣板（v1.7） |
| `/var/lib/rancher/rke2/agent/etc/containerd/config-v3.toml.tmpl` | containerd 自訂樣板（v2.0+） |
| `/var/lib/rancher/rke2/bin/` | 內建 `kubectl`、`crictl`、`ctr` 等工具 |
| `/etc/default/rke2-server` / `/etc/default/rke2-agent` | 環境變數檔（systemd EnvironmentFile） |

---

## 6. 與 NVIDIA Run:ai 的整合切入點

Run:ai 在 RKE2 上的部署位置：

- **GPU 節點（HGX 實體機）**：（NVIDIA Driver 建議交由 GPU Operator 部署）→ 加入 RKE2 → 部署 **NVIDIA GPU Operator**（containerd 整合 nvidia runtime）→ 部署 **Run:ai cluster**。
- **Ingress**：RKE2 v1.35 預設提供 ingress-nginx（Run:ai 需要 Ingress Controller，RKE2 已預載）。
- **Prometheus**：Run:ai 必要前置元件，**需自行安裝**（RKE2 預設只有 metrics-server，不含 Prometheus）。
- **Namespace**：Run:ai 需要 `runai` namespace。

> 詳細整合流程不在本 RKE2 文件範圍內；本文件只覆蓋 RKE2 與 GPU Operator 之間的銜接點。Run:ai 部署細節請參考 [Run:ai 官方文件](https://run-ai-docs.nvidia.com/)。
