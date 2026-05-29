# 05 – 叢集驗證

> 來源：
> - [https://docs.rke2.io/install/quickstart](https://docs.rke2.io/install/quickstart)
> - [https://docs.rke2.io/add-ons/gpu_operators](https://docs.rke2.io/add-ons/gpu_operators)
> 整理日期：2026-05-26
> 目標版本：RKE2 v1.35.x（stable channel）

---

## 1. 節點層級驗證

### 1.1 服務狀態

```bash
# Server
sudo systemctl status rke2-server
sudo journalctl -u rke2-server --since "10 min ago" | grep -iE 'error|fail'

# Agent
sudo systemctl status rke2-agent
sudo journalctl -u rke2-agent --since "10 min ago" | grep -iE 'error|fail'
```

### 1.2 內建工具路徑

```bash
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

kubectl version
crictl version
ctr version
```

---

## 2. 叢集層級驗證

### 2.1 節點與系統 Pod

```bash
kubectl get nodes -o wide
# 所有節點 STATUS=Ready

kubectl get pods -A -o wide
# 重點關注 kube-system / gpu-operator namespace 無 CrashLoopBackOff
```

### 2.2 預期的系統 Pod 清單（v1.35.x stable）

| Namespace | Pod 前綴 | 角色 |
|-----------|----------|------|
| `kube-system` | `etcd-*` | etcd（每 Server 1 個） |
| `kube-system` | `kube-apiserver-*` | API server |
| `kube-system` | `kube-controller-manager-*` | Controller |
| `kube-system` | `kube-scheduler-*` | Scheduler |
| `kube-system` | `cloud-controller-manager-*` | Cloud provider（若無 cloud-provider 設定，部分版本下不一定顯示，**待驗證**） |
| `kube-system` | `kube-proxy-*` | Service proxy（每節點 1 個） |
| `kube-system` | `rke2-canal-*` | CNI（DaemonSet） |
| `kube-system` | `rke2-coredns-*` | DNS |
| `kube-system` | `rke2-ingress-nginx-controller-*` | Ingress（v1.35 預設） |
| `kube-system` | `rke2-metrics-server-*` | Metrics |
| `kube-system` | `helm-install-*` | 一次性 Helm 部署 job（Completed） |
| `gpu-operator` | `nvidia-*`（多個） | GPU Operator（僅 GPU 節點） |

---

## 3. etcd 健康檢查

來源：[https://docs.rke2.io/datastore/backup_restore](https://docs.rke2.io/datastore/backup_restore)

```bash
# 從任一 Server 節點
kubectl -n kube-system exec etcd-rke2-srv-01 -- etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health --cluster

# 預期輸出
# https://192.168.10.11:2379 is healthy: ...
# https://192.168.10.12:2379 is healthy: ...
# https://192.168.10.13:2379 is healthy: ...

# etcd 成員列表
kubectl -n kube-system exec etcd-rke2-srv-01 -- etcdctl \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  member list
```

---

## 4. 網路功能驗證

### 4.1 Pod ↔ Pod 跨節點通訊

```bash
# 建立兩個分別在不同節點的測試 Pod
kubectl run net-test-1 --image=nicolaka/netshoot --restart=Never \
  --overrides='{"spec":{"nodeName":"rke2-wkr-01"}}' -- sleep 3600

kubectl run net-test-2 --image=nicolaka/netshoot --restart=Never \
  --overrides='{"spec":{"nodeName":"rke2-wkr-02"}}' -- sleep 3600

# 取得 Pod IP
POD2_IP=$(kubectl get pod net-test-2 -o jsonpath='{.status.podIP}')

# 從 Pod 1 ping Pod 2
kubectl exec net-test-1 -- ping -c 3 $POD2_IP
```

### 4.2 Pod → Service 解析

```bash
kubectl exec net-test-1 -- nslookup kubernetes.default.svc.cluster.local
# 應解析至 service-cidr 內 IP（預設 10.43.0.1）
```

### 4.3 CoreDNS 驗證

```bash
kubectl get svc -n kube-system rke2-coredns-rke2-coredns
# CLUSTER-IP 應為 10.43.0.10（預設）
```

### 4.4 清理

```bash
kubectl delete pod net-test-1 net-test-2
```

---

## 5. Ingress 驗證（v1.35 預設為 ingress-nginx）

```bash
kubectl get pods -n kube-system | grep ingress
kubectl get svc -n kube-system | grep ingress
# 服務型別預設為 LoadBalancer 或 NodePort，依環境設定
```

部署測試應用：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
spec:
  replicas: 1
  selector:
    matchLabels: {app: hello-web}
  template:
    metadata: {labels: {app: hello-web}}
    spec:
      containers:
      - name: web
        image: nginxdemos/hello
        ports: [{containerPort: 80}]
---
apiVersion: v1
kind: Service
metadata: {name: hello-web}
spec:
  selector: {app: hello-web}
  ports: [{port: 80, targetPort: 80}]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: hello.example.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-web
            port: {number: 80}
```

```bash
kubectl apply -f hello-web.yaml
curl -H "Host: hello.example.local" http://<任一節點 IP>/
```

---

## 6. GPU 資源驗證

```bash
# 1. GPU 節點標籤
kubectl get nodes -L workload,nvidia.com/gpu.product

# 2. GPU 資源可分配量
kubectl get node rke2-gpu-01 -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'
# 8（HGX H100/H200 通常 8）

# 3. nvidia-smi 在 Pod 中執行
kubectl run nvidia-smi --rm -it --restart=Never \
  --image=nvcr.io/nvidia/cuda:12.4.0-base-ubuntu22.04 \
  --overrides='{"spec":{"tolerations":[{"key":"nvidia.com/gpu","operator":"Equal","value":"true","effect":"NoSchedule"}],"containers":[{"name":"nvidia-smi","image":"nvcr.io/nvidia/cuda:12.4.0-base-ubuntu22.04","command":["nvidia-smi"],"resources":{"limits":{"nvidia.com/gpu":1}}}]}}'
```

> **CUDA 鏡像版本**：本範例使用 12.4，**待驗證**：依 GPU Operator 部署的 driver 版本選用對應 CUDA tag。

---

## 7. 完整端對端煙霧測試（Smoke Test）

```bash
# 1. 部署一個有 1 個 replica 的應用，繫結到 GPU 節點
kubectl apply -f gpu-test-pod.yaml   # 參見 03-RKE2-Worker-Node-Setup.md §3.4

# 2. 確認 Pod 成功取得 GPU
kubectl logs nbody-gpu-benchmark

# 3. 觸發 HA failover：關閉 srv-01，驗證 API 仍可使用
sudo systemctl stop rke2-server   # 於 srv-01 執行
kubectl get nodes   # 從外部仍可查詢

# 4. 復原 srv-01
sudo systemctl start rke2-server
kubectl get nodes   # srv-01 重新 Ready
```

---

## 8. 驗證清單

- [ ] 6 個節點（3 server + 2 cpu worker + 1 gpu worker）皆 Ready
- [ ] 所有 kube-system Pod 為 Running / Completed
- [ ] etcd `endpoint health --cluster` 3 個成員 healthy
- [ ] Pod 跨節點通訊正常
- [ ] CoreDNS 解析 `kubernetes.default` 成功
- [ ] Ingress 測試應用可從外部 curl 取得回應
- [ ] GPU 節點 `nvidia.com/gpu` 資源 > 0
- [ ] 測試 GPU Pod 成功執行 nvidia-smi / nbody
- [ ] HA failover 測試通過
