# 06 – etcd 備份與災難還原

> 來源：
> - [https://docs.rke2.io/datastore/backup_restore](https://docs.rke2.io/datastore/backup_restore)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. 概念

RKE2 使用 **embedded etcd**（內建於 server 節點）。叢集狀態的所有資訊存於 etcd，**etcd 的備份等於叢集的備份**。

### 1.1 兩種快照來源

| 類型 | 觸發方式 | 預設行為 |
|------|---------|----------|
| **自動快照** | systemd timer | 每日 00:00、12:00（cron `0 */12 * * *`），保留 5 份 |
| **手動快照** | `rke2 etcd-snapshot save` | 立即執行；用於升級／重大變更前 |

### 1.2 兩種儲存位置

| 位置 | 適用情境 |
|------|----------|
| 本機檔案系統 `/var/lib/rancher/rke2/server/db/snapshots/` | 預設；單節點災難復原（其他節點仍存在） |
| **S3 相容物件儲存** | 災難復原（整個 site 失效） |

---

## 2. 自動快照設定

來源：[https://docs.rke2.io/datastore/backup_restore](https://docs.rke2.io/datastore/backup_restore)

### 2.1 主要參數（寫入 `/etc/rancher/rke2/config.yaml`）

| 參數 | 預設 | 說明 |
|------|------|------|
| `etcd-snapshot-schedule-cron` | `0 */12 * * *` | cron 排程 |
| `etcd-snapshot-retention` | `5` | 保留份數 |
| `etcd-snapshot-dir` | `${data-dir}/db/snapshots` | 本機儲存目錄 |
| `etcd-snapshot-compress` | `false` | 啟用 gzip 壓縮 |
| `etcd-disable-snapshots` | `false` | 停用自動快照 |

範例（每 6 小時、保留 14 份、啟用壓縮）：

```yaml
# /etc/rancher/rke2/config.yaml（每個 server 節點）
etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 14
etcd-snapshot-compress: true
```

修改後重啟：`sudo systemctl restart rke2-server`。

### 2.2 S3 上傳設定

```yaml
etcd-s3: true
etcd-s3-endpoint: "s3.example.com"
etcd-s3-access-key: "AKIAxxx"
etcd-s3-secret-key: "xxxxxx"
etcd-s3-bucket: "rke2-backups"
etcd-s3-region: "us-east-1"
etcd-s3-folder: "prod-cluster"
etcd-s3-retention: 30
```

> **安全建議**：
> - 不要將 S3 secret key 直接寫入 `config.yaml`（檔案權限）。可改用環境變數 `RKE2_ETCD_S3_*` 寫入 `/etc/default/rke2-server`，並將該檔權限設為 600。
> - S3 桶啟用 versioning + lifecycle policy，避免快照被誤刪。

---

## 3. 手動快照

```bash
# 建立手動快照（執行 RKE2 仍在運行時）
sudo /var/lib/rancher/rke2/bin/rke2 etcd-snapshot save \
  --name pre-upgrade-$(date +%Y%m%d-%H%M)

# 列出所有快照
sudo /var/lib/rancher/rke2/bin/rke2 etcd-snapshot list

# 刪除指定快照
sudo /var/lib/rancher/rke2/bin/rke2 etcd-snapshot delete <snapshot-name>

# 依保留數修剪舊快照
sudo /var/lib/rancher/rke2/bin/rke2 etcd-snapshot prune --snapshot-retention 10
```

> **任何升級、重大設定變更、Run:ai 大版本更新前，必須先手動快照**。

---

## 4. Token 備份（**還原前提**）

來源：[https://docs.rke2.io/datastore/backup_restore](https://docs.rke2.io/datastore/backup_restore)

> "You must also back up the server token file at `/var/lib/rancher/rke2/server/token`... the token is used to encrypt confidential data within the datastore itself."

**Token 必須與 etcd 快照一起備份**。建議定期執行：

```bash
# 例：每日 cron
sudo cp /var/lib/rancher/rke2/server/token \
        /backup/rke2-token-$(date +%Y%m%d).bak
sudo chmod 600 /backup/rke2-token-*.bak
```

並將備份檔案放到與 etcd 快照**不同位置**（例如 S3 加密儲存桶 + 異地 NAS）。

> **後果**：若 token 遺失，**所有 etcd 快照皆無法還原**，相當於資料遺失。

---

## 5. 還原流程

### 5.1 還原情境分類

| 情境 | 處理 |
|------|------|
| 單一 Server 故障（其餘 2 個 etcd 健康） | **不需還原**；刪除故障節點後重新加入即可 |
| 全部 Server 故障 / etcd 損毀 | 使用 [§5.2](#52-多-server-叢集還原從快照) 流程 |
| 整個叢集遺失（含實體機） | 使用 [§5.3](#53-跨環境還原從-s3-至新環境) 流程 |

### 5.2 多 Server 叢集還原（從快照）

來源：[https://docs.rke2.io/datastore/backup_restore](https://docs.rke2.io/datastore/backup_restore)

**第 1 步：在 srv-01（主節點）執行 cluster-reset**

```bash
sudo systemctl stop rke2-server

sudo /var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/<snapshot-name>

# 完成後會自動結束。重新啟動：
sudo systemctl start rke2-server

# 確認單節點叢集恢復
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
kubectl get nodes
```

**第 2 步：在 srv-02、srv-03 清空 etcd 資料並重新加入**

```bash
# 於 srv-02 與 srv-03 各自執行
sudo systemctl stop rke2-server
sudo rm -rf /var/lib/rancher/rke2/server/db/
sudo systemctl start rke2-server
sudo journalctl -u rke2-server -f
```

**第 3 步：驗證**

```bash
kubectl get nodes   # 3 個 server 重新 Ready
kubectl -n kube-system get pods   # 系統 pod 全部 Running
```

### 5.3 跨環境還原（從 S3 至新環境）

當原叢集硬體完全失效，需在新硬體重建：

**第 1 步：準備新主機（最少 1 個 server）**

依 [01-Prerequisites.md](01-RKE2-Prerequisites.md) 與 [02-Installation-HA.md](02-RKE2-Installation-HA.md) 完成 OS、防火牆、套件等前置設定。**安裝 RKE2 後不要啟動 service**。

**第 2 步：放回備份的 token**

```bash
sudo mkdir -p /var/lib/rancher/rke2/server
sudo cp /backup/rke2-token-YYYYMMDD.bak /var/lib/rancher/rke2/server/token
sudo chmod 600 /var/lib/rancher/rke2/server/token
```

**第 3 步：執行還原（從 S3）**

```bash
# 將 <原 token 字串> 替換為原叢集的 token（必須完全相同，否則無法解密快照）
sudo /var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --token='<原 token 字串>' \
  --etcd-s3 \
  --etcd-s3-endpoint=s3.example.com \
  --etcd-s3-bucket=rke2-backups \
  --etcd-s3-folder=prod-cluster \
  --etcd-s3-access-key=AKIAxxx \
  --etcd-s3-secret-key=xxxxxx \
  --cluster-reset-restore-path=<快照名稱>
```

> **token 驗證**：用於解密快照中的機密資料。token 可透過以下任一方式提供，兩者均有效：
> - `--token=<值>` CLI 旗標（如上方指令所示）
> - 預先寫入 `/etc/rancher/rke2/config.yaml` 的 `token:` 欄位（若已依第 4 步完成設定，可省略 CLI 旗標）
>
> **token 必須與原叢集完全相同**，否則快照無法解密使用（[來源](https://docs.rke2.io/datastore/backup_restore)）。

**第 4 步：寫入 config.yaml 並啟動**

```bash
sudo tee /etc/rancher/rke2/config.yaml <<EOF
token: mySharedSecretToken   # 必須與原 token 相同
tls-san:
  - rke2.example.local
  - 192.168.10.10
EOF

sudo systemctl enable rke2-server
sudo systemctl start rke2-server
```

**第 5 步：加入其他 server 與 worker**

依 [02-Installation-HA.md §3.2](02-RKE2-Installation-HA.md) 流程，使用同一個 token。

> **重要**：跨環境還原時，原有的 Worker 節點記錄仍在 etcd。若新環境節點 IP / 主機名不同，需先 `kubectl delete node <old-name>`，否則會看到大量 NotReady 殘留記錄。

### 5.4 從本機快照還原（已設定 S3 但只想用本機檔）

```bash
sudo /var/lib/rancher/rke2/bin/rke2 server \
  --cluster-reset \
  --etcd-s3=false \
  --cluster-reset-restore-path=/path/to/local/snapshot
```

`--etcd-s3=false` 強制覆蓋 config 中的 `etcd-s3: true` 設定。

---

## 6. 還原前後檢核

### 6.1 還原前

- [ ] 確認快照檔案完整可讀（`file <snapshot>` 確認為 zstd / gzip / etcd snapshot）
- [ ] 確認 token 已備份且可讀
- [ ] 確認所有 server 已停止 `rke2-server`（多節點還原避免 split-brain）
- [ ] 通知相關人員：API 將短暫不可用

### 6.2 還原後

- [ ] `kubectl get nodes` 全部 Ready
- [ ] `kubectl get pods -A` 系統 namespace 無錯誤
- [ ] etcd 健康（[05-Validation.md §3](05-RKE2-Validation.md)）
- [ ] 應用工作負載仍存在且正常
- [ ] 重新啟用自動快照（檢查 `etcd-disable-snapshots` 未誤啟用）

---

## 7. 備份政策建議

| 項目 | 建議 |
|------|------|
| 頻率 | 每 6 小時自動 + 重大變更前手動 |
| 保留 | 本機 14 份 + S3 30 份 |
| 異地 | S3 啟用跨區複製 |
| Token | 每日複製到備份系統，至少保留 30 份歷史 |
| 演練 | 每季在非生產環境執行一次完整還原演練 |
| 監控 | 監控 snapshot 檔案最新時間（超過 24h 無新快照 → 告警） |
