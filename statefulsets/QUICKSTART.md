# 本地 K8s 集群快速开始 / Local K8s Quick Start

## 中文版

### 在本地集群快速部署 StatefulSet

如果你使用 k3d、kind、minikube 或 k3s 等本地 Kubernetes 集群，可以使用以下命令一键部署：

```bash
cd statefulsets
kubectl apply -f local-app.yml
```

查看部署状态：
```bash
kubectl get statefulset tkb-sts
kubectl get pods -l app=web
kubectl get pvc
```

详细说明请参阅 [statefulsets/README.md](statefulsets/README.md)

---

## English Version

### Quick Deploy StatefulSet on Local Cluster

If you are using a local Kubernetes cluster like k3d, kind, minikube, or k3s, you can deploy with one command:

```bash
cd statefulsets
kubectl apply -f local-app.yml
```

Check deployment status:
```bash
kubectl get statefulset tkb-sts
kubectl get pods -l app=web
kubectl get pvc
```

For detailed instructions, see [statefulsets/README-EN.md](statefulsets/README-EN.md)
