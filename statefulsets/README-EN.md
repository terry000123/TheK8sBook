# StatefulSets - Local Deployment Guide

This directory contains example configuration files for StatefulSets. Below are instructions on how to run these examples on a local Kubernetes cluster.

## File Descriptions

### Cloud Platform Configuration Files
- `gcp-sc.yml` - Google Cloud Platform (GKE) StorageClass configuration
- `dd-sc.yml` - Docker Desktop StorageClass configuration
- `app.yml` - Complete GCP deployment (includes StorageClass, Service, and StatefulSet)

### Local Cluster Configuration Files
- `local-sc.yml` - Local Kubernetes cluster StorageClass (for k3d, kind, minikube, k3s)
- `local-app.yml` - Complete local deployment (includes StorageClass, Service, and StatefulSet)

### General Configuration Files
- `sts.yml` - StatefulSet configuration (requires pre-existing StorageClass)
- `headless-svc.yml` - Headless Service configuration
- `jump-pod.yml` - Jump pod for testing

## Deploying on Local K8s Cluster

### Prerequisites

Ensure your local Kubernetes cluster has local-path-provisioner installed. Most local Kubernetes distributions include it by default:

- **k3d/k3s**: Includes `rancher.io/local-path` provisioner by default
- **kind**: Includes `standard` storage class (hostPath) by default
- **minikube**: Can use `standard` storage class
- **Docker Desktop**: Uses `docker.io/hostpath` provisioner

### Method 1: Use Complete Local Configuration File (Recommended)

This is the easiest method, deploying all resources at once:

```bash
# Deploy all resources (StorageClass, Service, and StatefulSet)
kubectl apply -f local-app.yml

# Check StatefulSet status
kubectl get statefulset tkb-sts

# Check Pods
kubectl get pods -l app=web

# Check PVCs
kubectl get pvc
```

### Method 2: Step-by-Step Deployment

If you want to understand each component better:

```bash
# 1. Create StorageClass
kubectl apply -f local-sc.yml

# 2. Create Headless Service
kubectl apply -f headless-svc.yml

# 3. Create StatefulSet
kubectl apply -f sts.yml

# View resources
kubectl get storageclass flash
kubectl get service dullahan
kubectl get statefulset tkb-sts
kubectl get pods -l app=web
kubectl get pvc
```

### For Different Local Clusters

#### k3d/k3s
```bash
# k3d/k3s use rancher.io/local-path by default
kubectl apply -f local-app.yml
```

#### kind
```bash
# kind uses standard storage class, may need to modify storage class name
# Option 1: Use default standard storage class
kubectl apply -f local-app.yml
# Then modify PVC to use "standard" instead of "flash"

# Option 2: Create alias
kubectl apply -f local-sc.yml
```

#### minikube
```bash
# minikube typically uses standard storage class
kubectl apply -f local-app.yml
```

#### Docker Desktop
```bash
# Docker Desktop users can use dedicated configuration
kubectl apply -f dd-sc.yml
kubectl apply -f headless-svc.yml
kubectl apply -f sts.yml
```

## Verify Deployment

### Check StatefulSet Status

```bash
# View StatefulSet
kubectl get statefulset tkb-sts

# View Pods (should see tkb-sts-0, tkb-sts-1, tkb-sts-2)
kubectl get pods -l app=web -o wide

# View PVCs (each Pod has its own PVC)
kubectl get pvc

# View PVs
kubectl get pv
```

### Test Pod DNS and Storage

```bash
# Deploy jump pod for testing
kubectl apply -f jump-pod.yml

# Wait for Pod to be ready
kubectl wait --for=condition=ready pod/jump-pod --timeout=60s

# Test DNS resolution (StatefulSet Pods have stable DNS names)
kubectl exec -it jump-pod -- nslookup tkb-sts-0.dullahan
kubectl exec -it jump-pod -- nslookup tkb-sts-1.dullahan
kubectl exec -it jump-pod -- nslookup tkb-sts-2.dullahan

# Test HTTP access
kubectl exec -it jump-pod -- curl tkb-sts-0.dullahan
```

### Test Persistent Storage

```bash
# Write data in first Pod
kubectl exec tkb-sts-0 -- sh -c 'echo "Hello from tkb-sts-0" > /usr/share/nginx/html/index.html'

# Verify data
kubectl exec jump-pod -- curl tkb-sts-0.dullahan

# Delete Pod (StatefulSet will recreate it)
kubectl delete pod tkb-sts-0

# Wait for Pod to be recreated
kubectl wait --for=condition=ready pod/tkb-sts-0 --timeout=60s

# Verify data still exists (persistence successful)
kubectl exec jump-pod -- curl tkb-sts-0.dullahan
```

## Cleanup Resources

```bash
# Delete StatefulSet (but keep PVCs)
kubectl delete statefulset tkb-sts

# Delete Service
kubectl delete service dullahan

# Delete PVCs (this will delete data)
kubectl delete pvc -l app=web

# Delete jump pod
kubectl delete pod jump-pod

# Delete StorageClass (if needed)
kubectl delete storageclass flash
```

Or delete all resources at once:

```bash
kubectl delete -f local-app.yml
kubectl delete pod jump-pod
```

## Common Issues

### PVC Stuck in Pending State

If PVC remains in Pending state:

1. Check if storage class is available:
   ```bash
   kubectl get storageclass
   ```

2. Check if local-path-provisioner is installed (k3d/k3s):
   ```bash
   kubectl get pods -n kube-system | grep local-path
   ```

3. For kind, ensure correct storage class name is used:
   ```bash
   kubectl get storageclass
   # Use the default storage class name shown
   ```

### Pod Won't Start

Check Pod events:
```bash
kubectl describe pod tkb-sts-0
```

Common causes:
- StorageClass doesn't exist or misconfigured
- Node doesn't have enough storage
- PVC binding failed

### Modify Storage Class Name

If your cluster uses a different storage class name (e.g., "standard" instead of "flash"), edit the configuration file:

```bash
# View available storage classes
kubectl get storageclass

# Edit local-app.yml, change storageClassName: "flash" to your storage class name
```

## Learn More

- [Kubernetes StatefulSets Official Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Persistent Volumes Official Documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)
