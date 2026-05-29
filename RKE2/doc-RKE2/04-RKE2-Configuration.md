# 04 – RKE2 設定參考（config.yaml / 環境變數 / 進階）

> 來源：
> - [https://docs.rke2.io/install/configuration](https://docs.rke2.io/install/configuration)
> - [https://docs.rke2.io/reference/server_config](https://docs.rke2.io/reference/server_config)
> - [https://docs.rke2.io/advanced](https://docs.rke2.io/advanced)
> - [https://docs.rke2.io/networking/basic_network_options](https://docs.rke2.io/networking/basic_network_options)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. 設定檔機制

### 1.1 檔案位置

| 檔案 | 作用 |
|------|------|
| `/etc/rancher/rke2/config.yaml` | 主設定檔 |
| `/etc/rancher/rke2/config.yaml.d/*.yaml` | 子設定檔，依**字母序**合併（後者覆寫前者） |

### 1.2 設定優先序

1. 命令列旗標（最高優先）
2. 環境變數（`RKE2_*`、`INSTALL_RKE2_*`）
3. 主設定檔 `/etc/rancher/rke2/config.yaml`
4. 子設定檔 `config.yaml.d/*.yaml`（字母序）

### 1.3 多檔案合併範例

來源：[https://docs.rke2.io/install/configuration](https://docs.rke2.io/install/configuration)

```yaml
# /etc/rancher/rke2/config.yaml
token: boop
node-label:
  - foo=bar
  - bar=baz

# /etc/rancher/rke2/config.yaml.d/test1.yaml
write-kubeconfig-mode: 600
node-taint:
  - alice=bob:NoExecute

# /etc/rancher/rke2/config.yaml.d/test2.yaml
write-kubeconfig-mode: 777
node-label:
  - other=what
  - foo=three        # 覆蓋 foo=bar
node-taint+:        # 「+」表示追加而非覆蓋
  - charlie=delta:NoSchedule
```

> **`+` 後綴**：用於 list 型參數時表示「追加」，不加則「覆蓋」。

---

## 2. Server 常用參數

來源：[https://docs.rke2.io/reference/server_config](https://docs.rke2.io/reference/server_config)

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `token` | （無，自動產生） | 節點加入共用 secret |
| `tls-san` | （無） | 額外 SAN（LB FQDN / VIP） |
| `cluster-cidr` | `10.42.0.0/16` | Pod IP 範圍 |
| `service-cidr` | `10.43.0.0/16` | Service IP 範圍 |
| `cluster-dns` | `10.43.0.10` | CoreDNS Service IP |
| `cluster-domain` | `cluster.local` | DNS suffix |
| `service-node-port-range` | `30000-32767` | NodePort 範圍 |
| `cni` | `canal` | CNI plugin |
| `disable` | （無） | 停用內建 chart（見 [§4](#4-停用內建元件)） |
| `write-kubeconfig-mode` | （無） | kubeconfig 檔案權限（建議 `0640` 或 `0600`） |
| `node-label` | （無） | 節點標籤 |
| `node-taint` | （無） | 節點 taint |
| `profile` | （無） | `cis`（CIS Benchmark 模式） |
| `debug` | `false` | 詳細日誌 |
| `selinux` | `false` | SELinux 強制模式（RHEL 系） |
| `kubelet-arg` | （無） | 透傳 kubelet 旗標 |
| `kube-apiserver-arg` | （無） | 透傳 apiserver 旗標 |

> **HA 必須一致**：`cluster-cidr`、`service-cidr`、`cluster-dns`、`cluster-domain`、`service-node-port-range`、`cni`、`disable`、`profile` 在**所有 server** 上必須相同。

---

## 3. CNI 選擇

來源：[https://docs.rke2.io/networking/basic_network_options](https://docs.rke2.io/networking/basic_network_options)

### 3.1 內建 CNI 對照

| CNI | 適用情境 | 備註 |
|-----|----------|------|
| **Canal**（Calico + Flannel） | 預設、通用 | 簡單可靠；本案建議 |
| **Cilium** | eBPF、進階網路政策、kube-proxy replacement | 學習曲線陡；功能強大 |
| **Calico** | BGP 整合、進階網路政策 | 需處理 BGP peer |
| **Flannel** | 最輕量 | 功能較少；Windows 節點可用 |
| **Multus** | 多 CNI 並存（Pod 多網卡） | 必須搭配 primary CNI |

### 3.2 切換 CNI

```yaml
# config.yaml
cni: cilium
```

> **變更 CNI 後**：需重建叢集（不能熱切換）。生產環境應在初次部署時決定。

### 3.3 Dual-stack（IPv4 + IPv6）

```yaml
cluster-cidr: "10.42.0.0/16,2001:cafe:42::/56"
service-cidr: "10.43.0.0/16,2001:cafe:43::/112"
```

---

## 4. 停用內建元件

來源：[https://docs.rke2.io/reference/server_config](https://docs.rke2.io/reference/server_config)

可停用的內建 Helm chart：

| 元件名稱 | 預設 | 停用情境 |
|----------|------|----------|
| `rke2-coredns` | 啟用 | 若用外部 DNS（如 NodeLocal DNSCache 自行管理） |
| `rke2-ingress-nginx` | v1.35 預設啟用，v1.36 預設改 Traefik | 若使用 Run:ai 內含 ingress 或自行部署 Traefik / Istio |
| `rke2-metrics-server` | 啟用 | 若部署完整 Prometheus stack 取代 |
| `rke2-snapshot-controller` / `rke2-snapshot-validation-webhook` | 啟用 | 若不使用 CSI snapshot 功能 |

範例：

```yaml
disable:
  - rke2-ingress-nginx
  - rke2-metrics-server
```

> **Run:ai 整合考量**：Run:ai 需要 Ingress Controller，若叢集無其他選擇，**勿停用 ingress-nginx**。

---

## 5. 自訂內建 Helm Chart

來源：[https://docs.rke2.io/advanced](https://docs.rke2.io/advanced)

於 `/var/lib/rancher/rke2/server/manifests/` 建立 `HelmChartConfig`，例如自訂 Canal：

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-canal
  namespace: kube-system
spec:
  valuesContent: |-
    calico:
      vethuMTU: 1450
```

> **注意**：`HelmChartConfig` 的 `name` 必須與內建 chart 名稱一致；放錯目錄不會生效。

---

## 6. Private Registry 設定

來源：[https://docs.rke2.io/install/airgap](https://docs.rke2.io/install/airgap)

於每個節點建立 `/etc/rancher/rke2/registries.yaml`：

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://registry.internal.example.com"
  "registry.internal.example.com":
    endpoint:
      - "https://registry.internal.example.com"
configs:
  "registry.internal.example.com":
    auth:
      username: <實際帳號>     # registries.yaml 不會做 shell 變數展開，請直接填入
      password: <實際密碼>
    tls:
      ca_file: /etc/rancher/rke2/registry-ca.crt
```

> **重要**：
> - `registries.yaml` 為純靜態 YAML，**不會展開 `${VAR}` 環境變數**；必須直接寫入值。
> - 此檔含明文密碼，**不要提交至 git**；建議檔案權限設為 `0600`，僅 root 可讀。
> - 設定後**重啟 rke2 服務**才會載入。

---

## 7. 自訂 containerd 設定

來源：[https://docs.rke2.io/advanced](https://docs.rke2.io/advanced)

> **不要直接編輯 `config.toml`**（會被覆蓋）。改放樣板：

| containerd 版本 | 樣板路徑 |
|-----------------|---------|
| containerd 2.0+ | `/var/lib/rancher/rke2/agent/etc/containerd/config-v3.toml.tmpl` |
| containerd 1.7 及以下 | `/var/lib/rancher/rke2/agent/etc/containerd/config.toml.tmpl` |

樣板使用 Go `text/template` 語法。複製預設 config 作為起點：

```bash
sudo cp /var/lib/rancher/rke2/agent/etc/containerd/config.toml \
        /var/lib/rancher/rke2/agent/etc/containerd/config-v3.toml.tmpl
# 編輯後重啟
sudo systemctl restart rke2-agent
```

---

## 8. 控制平面元件自訂

來源：[https://docs.rke2.io/advanced](https://docs.rke2.io/advanced)

RKE2 提供以下系列旗標將額外參數透傳給控制平面 static pod：

| 旗標系列 | 用途 |
|----------|------|
| `kube-apiserver-arg` | 透傳 args 給 kube-apiserver |
| `kube-apiserver-extra-env` | 加入環境變數 |
| `kube-apiserver-extra-mount` | 加入 volume mount |
| `kube-controller-manager-arg` / `-extra-env` / `-extra-mount` | 對應 controller-manager |
| `kube-scheduler-arg` / `-extra-env` / `-extra-mount` | 對應 scheduler |
| `etcd-arg` / `-extra-env` / `-extra-mount` | 對應 etcd |
| `kubelet-arg` | 透傳給 kubelet（非 static pod，host 進程） |

範例（啟用 audit log）：

```yaml
# config.yaml
kube-apiserver-extra-mount:
  - "/var/log/audit:/var/log/audit"
kube-apiserver-arg:
  - "audit-log-path=/var/log/audit/kube-audit.log"
  - "audit-log-maxsize=100"
```

> 完整參數請對照 [Server Config](https://docs.rke2.io/reference/server_config) 與 [Agent Config](https://docs.rke2.io/reference/agent_config)。

---

## 9. 環境變數檔案

systemd 服務從以下檔案載入環境變數：

| 角色 | 檔案 |
|------|------|
| Server | `/etc/default/rke2-server`（Ubuntu） |
| Agent | `/etc/default/rke2-agent`（Ubuntu） |

> 修改後須 `systemctl daemon-reload && systemctl restart rke2-*`。

---

## 10. 本案完整 config.yaml 範例

### 10.1 Server #1（rke2-srv-01，初始）

```yaml
# /etc/rancher/rke2/config.yaml
token: mySharedSecretToken
tls-san:
  - rke2.example.local
  - 192.168.10.10
write-kubeconfig-mode: "0640"
node-label:
  - role=control-plane
# cluster-cidr/service-cidr 使用預設；如需自訂須三 server 一致
```

### 10.2 Server #2、#3

```yaml
server: https://rke2.example.local:9345
token: mySharedSecretToken
tls-san:
  - rke2.example.local
  - 192.168.10.10
write-kubeconfig-mode: "0640"
node-label:
  - role=control-plane
```

### 10.3 CPU Worker

```yaml
server: https://rke2.example.local:9345
token: mySharedSecretToken
node-label:
  - workload=cpu
```

### 10.4 GPU Worker

```yaml
server: https://rke2.example.local:9345
token: mySharedSecretToken
node-label:
  - workload=gpu
node-taint:
  - nvidia.com/gpu=true:NoSchedule
```
