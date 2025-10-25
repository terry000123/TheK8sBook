# StatefulSets - 本地部署指南

本目录包含 StatefulSet 的示例配置文件。以下是如何在本地 Kubernetes 集群上运行这些示例的说明。

## 文件说明

### 云平台配置文件
- `gcp-sc.yml` - Google Cloud Platform (GKE) 存储类配置
- `dd-sc.yml` - Docker Desktop 存储类配置
- `app.yml` - 完整的 GCP 部署配置（包含 StorageClass、Service 和 StatefulSet）

### 本地集群配置文件
- `local-sc.yml` - 本地 Kubernetes 集群存储类配置（适用于 k3d、kind、minikube、k3s）
- `local-app.yml` - 完整的本地部署配置（包含 StorageClass、Service 和 StatefulSet）

### 通用配置文件
- `sts.yml` - StatefulSet 配置（需要预先创建存储类）
- `headless-svc.yml` - Headless Service 配置
- `jump-pod.yml` - 用于测试的跳板 Pod

## 在本地 K8s 集群上部署

### 前置条件

确保你的本地 Kubernetes 集群已经安装了 local-path-provisioner。大多数本地 Kubernetes 发行版都默认包含：

- **k3d/k3s**: 默认包含 `rancher.io/local-path` provisioner
- **kind**: 默认包含 `standard` storage class (hostPath)
- **minikube**: 可以使用 `standard` storage class
- **Docker Desktop**: 使用 `docker.io/hostpath` provisioner

### 方法 1: 使用完整的本地配置文件（推荐）

这是最简单的方法，一次性部署所有资源：

```bash
# 部署所有资源（StorageClass、Service 和 StatefulSet）
kubectl apply -f local-app.yml

# 查看 StatefulSet 状态
kubectl get statefulset tkb-sts

# 查看 Pods
kubectl get pods -l app=web

# 查看 PVC
kubectl get pvc
```

### 方法 2: 分步部署

如果你想更好地理解每个组件：

```bash
# 1. 创建存储类
kubectl apply -f local-sc.yml

# 2. 创建 Headless Service
kubectl apply -f headless-svc.yml

# 3. 创建 StatefulSet
kubectl apply -f sts.yml

# 查看资源
kubectl get storageclass flash
kubectl get service dullahan
kubectl get statefulset tkb-sts
kubectl get pods -l app=web
kubectl get pvc
```

### 针对不同的本地集群

#### k3d/k3s
```bash
# k3d/k3s 默认使用 rancher.io/local-path
kubectl apply -f local-app.yml
```

#### kind
```bash
# kind 使用 standard storage class，需要修改 storage class name
# 选项 1: 使用默认的 standard storage class
kubectl apply -f local-app.yml
# 然后修改 PVC 使用 "standard" 而不是 "flash"

# 选项 2: 创建别名
kubectl apply -f local-sc.yml
```

#### minikube
```bash
# minikube 通常使用 standard storage class
kubectl apply -f local-app.yml
```

#### Docker Desktop
```bash
# Docker Desktop 用户可以使用专门的配置
kubectl apply -f dd-sc.yml
kubectl apply -f headless-svc.yml
kubectl apply -f sts.yml
```

## 验证部署

### 检查 StatefulSet 状态

```bash
# 查看 StatefulSet
kubectl get statefulset tkb-sts

# 查看 Pods（应该看到 tkb-sts-0, tkb-sts-1, tkb-sts-2）
kubectl get pods -l app=web -o wide

# 查看 PVC（每个 Pod 都有自己的 PVC）
kubectl get pvc

# 查看 PV
kubectl get pv
```

### 测试 Pod DNS 和存储

```bash
# 部署跳板 Pod 用于测试
kubectl apply -f jump-pod.yml

# 等待 Pod 就绪
kubectl wait --for=condition=ready pod/jump-pod --timeout=60s

# 测试 DNS 解析（StatefulSet 的 Pod 有稳定的 DNS 名称）
kubectl exec -it jump-pod -- nslookup tkb-sts-0.dullahan
kubectl exec -it jump-pod -- nslookup tkb-sts-1.dullahan
kubectl exec -it jump-pod -- nslookup tkb-sts-2.dullahan

# 测试 HTTP 访问
kubectl exec -it jump-pod -- curl tkb-sts-0.dullahan
```

### 测试持久化存储

```bash
# 在第一个 Pod 中写入数据
kubectl exec tkb-sts-0 -- sh -c 'echo "Hello from tkb-sts-0" > /usr/share/nginx/html/index.html'

# 验证数据
kubectl exec jump-pod -- curl tkb-sts-0.dullahan

# 删除 Pod（StatefulSet 会重新创建）
kubectl delete pod tkb-sts-0

# 等待 Pod 重新创建
kubectl wait --for=condition=ready pod/tkb-sts-0 --timeout=60s

# 验证数据仍然存在（持久化成功）
kubectl exec jump-pod -- curl tkb-sts-0.dullahan
```

## 清理资源

```bash
# 删除 StatefulSet（但保留 PVC）
kubectl delete statefulset tkb-sts

# 删除 Service
kubectl delete service dullahan

# 删除 PVC（这会删除数据）
kubectl delete pvc -l app=web

# 删除跳板 Pod
kubectl delete pod jump-pod

# 删除 StorageClass（如果需要）
kubectl delete storageclass flash
```

或者一次性删除所有资源：

```bash
kubectl delete -f local-app.yml
kubectl delete pod jump-pod
```

## 常见问题

### PVC 处于 Pending 状态

如果 PVC 一直处于 Pending 状态：

1. 检查是否有可用的 storage class：
   ```bash
   kubectl get storageclass
   ```

2. 检查是否安装了 local-path-provisioner（k3d/k3s）：
   ```bash
   kubectl get pods -n kube-system | grep local-path
   ```

3. 对于 kind，确保使用正确的 storage class name：
   ```bash
   kubectl get storageclass
   # 使用显示的默认 storage class name
   ```

### Pod 无法启动

检查 Pod 事件：
```bash
kubectl describe pod tkb-sts-0
```

常见原因：
- 存储类不存在或配置错误
- 节点没有足够的存储空间
- PVC 绑定失败

### 修改 storage class name

如果你的集群使用不同的 storage class name（例如 "standard" 而不是 "flash"），编辑配置文件：

```bash
# 查看可用的 storage class
kubectl get storageclass

# 编辑 local-app.yml，将 storageClassName: "flash" 改为你的 storage class name
```

## 了解更多

- [Kubernetes StatefulSets 官方文档](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Persistent Volumes 官方文档](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)
