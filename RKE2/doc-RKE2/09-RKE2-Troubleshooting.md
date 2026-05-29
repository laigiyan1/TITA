# 09 – 疑難排解

> 來源：
> - [https://docs.rke2.io/known_issues](https://docs.rke2.io/known_issues)
> - [https://docs.rke2.io/advanced](https://docs.rke2.io/advanced)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. 排查通用流程

1. **確認服務狀態**：`systemctl status rke2-server` / `rke2-agent`
2. **檢視最近日誌**：`journalctl -u rke2-server --since "30 min ago" -p err`
3. **檢視 static pod 狀態**：`crictl ps -a | head -20`
4. **檢視叢集事件**：`kubectl get events -A --sort-by=.lastTimestamp | tail -50`
5. **若涉及網路**：`crictl logs <canal-pod-id>`、`ip a`、`iptables -L -n`

---

## 2. 啟動失敗類

### 2.1 `rke2-server` 服務無法啟動

```bash
sudo journalctl -u rke2-server -n 200 --no-pager
```

| 錯誤訊息 | 可能原因 | 對策 |
|---------|---------|------|
| `port 6443 already in use` | 其他服務佔用埠 | `ss -ltnp \| grep 6443`；停用衝突服務 |
| `failed to load token` | token 檔案缺失或損毀 | 檢查 `/var/lib/rancher/rke2/server/token` 權限與內容 |
| `failed to find data store` | 升級失敗導致 etcd 不一致 | 從快照還原（[06-Backup-and-Restore.md](06-RKE2-Backup-and-Restore.md)） |
| `BootstrapKey already used` | 多 server 同時 bootstrap | 確認僅 srv-01 為初始節點，srv-02/03 等待後再啟動 |

### 2.2 `rke2-agent` 無法 join

| 錯誤訊息 | 可能原因 | 對策 |
|---------|---------|------|
| `cluster CA certificate hash does not match` | token 不正確或叢集被重建 | 從 server `/var/lib/rancher/rke2/server/node-token` 重新取得 |
| `dial tcp ...:9345: i/o timeout` | LB / 防火牆未開 9345 | 檢查 LB；`nc -zv <server-ip> 9345` |
| `x509: certificate signed by unknown authority` | server tls-san 未含 LB FQDN | 修正 server `tls-san` 並重啟（[02-Installation-HA §5](02-RKE2-Installation-HA.md)） |
| `failed to find data store` | agent 重灌時殘留 | `sudo rm -rf /var/lib/rancher/rke2/agent/etc/`（**僅 agent**） |

---

## 3. 節點狀態類

### 3.1 Node NotReady

```bash
kubectl describe node <node-name> | grep -A 10 Conditions
```

常見 Condition 與處理：

| Condition | 訊息 | 對策 |
|-----------|------|------|
| `KubeletNotReady` | `network plugin returns error: cni plugin not initialized` | CNI pod 未啟動；`kubectl get pods -n kube-system -l k8s-app=canal` |
| `MemoryPressure True` | 記憶體不足 | 擴大節點 RAM 或調整 eviction threshold |
| `DiskPressure True` | 磁碟使用 > 85% | 清理 `/var/lib/containerd/`（謹慎）或擴大磁碟 |
| `PIDPressure True` | 過多 fork | 檢查是否有 leak 的 daemonset |

### 3.2 Pod 卡在 `ContainerCreating`

```bash
kubectl describe pod <pod-name>
```

| 事件訊息 | 對策 |
|---------|------|
| `failed to pull image` | Registry 不通；檢查 [registries.yaml](04-RKE2-Configuration.md#6-private-registry-設定) |
| `MountVolume.SetUp failed` | PVC / CSI 問題；檢查 storageclass |
| `failed to allocate for range 0: no IP addresses available` | Pod IP 耗盡（[§5](#5-網路類) Canal IP 耗盡） |
| `network plugin returns error` | CNI 異常；重啟 canal pod |

---

## 4. etcd 類

### 4.1 etcd member 失聯

```bash
kubectl -n kube-system exec etcd-rke2-srv-01 -- etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health --cluster
```

**症狀**：1 個 member unhealthy。

**對策**：
1. 確認該 server VM 狀態（網路、CPU、磁碟）
2. 若是暫時性問題（重開機）：等待自動恢復
3. 若 member 永久損壞：從 etcd 移除並重新加入：
   ```bash
   # 取得 member ID
   etcdctl member list
   # 移除
   etcdctl member remove <id>
   # 在問題節點重置 RKE2 並重新 join
   sudo /usr/local/bin/rke2-killall.sh
   sudo rm -rf /var/lib/rancher/rke2/server/db/
   sudo systemctl start rke2-server
   ```

### 4.2 etcd DB size 過大

預設 etcd DB 配額為 2 GB。超過時 etcd 進入唯讀模式。

```bash
# 查看實際大小
etcdctl endpoint status --write-out=table

# 手動 compact + defrag
REV=$(etcdctl endpoint status --write-out=json | jq -r '.[0].Status.header.revision')
etcdctl compact $REV
etcdctl defrag --cluster
```

### 4.3 etcd 效能不足

```bash
# 進入 etcd pod
kubectl -n kube-system exec -it etcd-rke2-srv-01 -- sh

# 執行 perf check
etcdctl check perf
```

> **VMware 部署常見問題**：vSphere thin provisioning / shared datastore 上 fsync 延遲可能高達數十 ms，導致 etcd warning。對策：使用 thick eager-zeroed VMDK + 低延遲儲存層。

---

## 5. 網路類

### 5.1 Canal Pod CrashLoopBackOff

來源：[https://docs.rke2.io/known_issues](https://docs.rke2.io/known_issues)

| 原因 | 對策 |
|------|------|
| firewalld / ufw 干擾 | 停用或開放 8472/UDP + 9099/TCP |
| NetworkManager 管理 cali*/flannel* 介面 | 寫 `/etc/NetworkManager/conf.d/rke2-canal.conf` |
| iptables binary 缺失 | `apt-get install iptables` |

### 5.2 Pod 跨節點不通

```bash
# 確認 VXLAN tunnel
ip -d link show flannel.1   # 應存在
# 確認 iptables 規則
sudo iptables -t nat -L -n | grep canal
```

常見原因：
- 節點間 UDP 8472 被阻擋
- MTU 不一致（VMware VXLAN 場景需特別檢查；Canal 預設 1450）

### 5.3 CoreDNS 解析失敗

```bash
# 驗證 CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod>

# 從測試 Pod 驗證
kubectl run dnstest --image=busybox:1.36 --restart=Never --rm -it -- nslookup kubernetes.default
```

對策：
- 確認 `/etc/resolv.conf` 內沒有 `127.0.0.53`（systemd-resolved stub）造成 CoreDNS upstream 死循環
- 若有，將 kubelet `--resolv-conf` 改指向 `/run/systemd/resolve/resolv.conf`

### 5.4 NetworkManager 干擾

來源：[https://docs.rke2.io/known_issues](https://docs.rke2.io/known_issues)

```bash
sudo tee /etc/NetworkManager/conf.d/rke2-canal.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
EOF
sudo systemctl reload NetworkManager
```

---

## 6. GPU 類

### 6.1 GPU 節點無 `nvidia.com/gpu` 資源

```bash
# 1. 確認 driver pod
kubectl logs -n gpu-operator -l app=nvidia-driver-daemonset

# 2. 確認 device-plugin
kubectl logs -n gpu-operator -l app=nvidia-device-plugin-daemonset

# 3. 確認節點 capability
kubectl describe node rke2-gpu-01 | grep nvidia
```

常見原因：
- nouveau 未停用 → driver pod 失敗
- containerd config 未注入 nvidia runtime（v25.10 以下）
- containerd 版本與 GPU Operator 不相容

### 6.2 GPU Pod 啟動失敗

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

| 訊息 | 對策 |
|------|------|
| `Failed to initialize NVML: Unknown Error` | nvidia driver 與 toolkit 版本不匹配 |
| `nvidia-smi: command not found` | nvidia runtime 未注入 |
| `Insufficient nvidia.com/gpu` | 該節點 GPU 已被其他 Pod 佔用；`kubectl get pods -A -o wide | grep gpu` |

### 6.3 GPU Operator daemonset 部署不完整

```bash
kubectl get daemonset -n gpu-operator
# 確認 DESIRED == READY
```

若有 Pod 卡住，依該 Pod 角色查日誌：driver、container-toolkit、device-plugin、dcgm-exporter。

---

## 7. 應用層常見問題

### 7.1 Ingress 不通

```bash
kubectl get pods -n kube-system | grep ingress
kubectl logs -n kube-system <ingress-nginx-controller-pod>
kubectl get ingress -A
```

- 確認 Ingress 有正確的 `ingressClassName`（v1.35 預設為 `nginx`）
- 確認 svc / endpoint 存在
- 確認外部流量到 NodePort / LoadBalancer 通

### 7.2 PVC Pending

```bash
kubectl get pvc -A
kubectl describe pvc <name>
```

- RKE2 **不預設提供 storageclass**；需自行部署 CSI driver（NFS、Longhorn、vSphere CSI）
- VMware 環境推薦 vSphere CSI driver

---

## 8. 緊急救援工具

| 工具 | 路徑 | 用途 |
|------|------|------|
| `rke2-killall.sh` | `/usr/local/bin/rke2-killall.sh` | 立即停止 RKE2 與所有 pod（不解除安裝） |
| `rke2-uninstall.sh` | `/usr/local/bin/rke2-uninstall.sh` | 完全移除 RKE2（**包含資料**） |
| `crictl` | `/var/lib/rancher/rke2/bin/crictl` | 容器層級互動 |
| `kubectl` | `/var/lib/rancher/rke2/bin/kubectl` | 叢集操作 |

> **`rke2-uninstall.sh` 是不可逆操作**，會刪除 `/var/lib/rancher/rke2/`、`/etc/rancher/rke2/`、所有 systemd unit、binary。執行前務必確認意圖。

---

## 9. 取得官方協助

1. **官方文件**：[https://docs.rke2.io/](https://docs.rke2.io/)
2. **GitHub Issues**：[https://github.com/rancher/rke2/issues](https://github.com/rancher/rke2/issues)
3. **Rancher Slack**：[https://rancher-users.slack.com/](https://rancher-users.slack.com/) `#rke2` 頻道
4. **SUSE 付費支援**：擁有 SUSE Rancher subscription 者可開 case

附上問題描述時建議包含：
- RKE2 版本（`rke2 --version`）
- OS 與 kernel 版本（`uname -a`）
- 完整 `journalctl -u rke2-server` 過去 30 min 日誌
- `kubectl get nodes -o wide` 與 `kubectl get pods -A`
- 拓撲圖（節點、LB、CNI、CIDR）
