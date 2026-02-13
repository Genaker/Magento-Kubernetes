# Quick Start Guide

Get Magento running on Kubernetes in 5 minutes!

## Prerequisites

- Kubernetes cluster running (minikube, kind, or cloud provider)
- kubectl installed and configured
- At least 8GB RAM and 4 CPU cores available

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/Genaker/Magento-Kubernetes.git
cd Magento-Kubernetes
```

### 2. Deploy Everything

```bash
# Deploy all resources at once
kubectl apply -k .
```

### 3. Wait for Pods to be Ready

```bash
# This may take 5-10 minutes depending on your cluster
kubectl wait --for=condition=ready pod --all -n magento --timeout=600s
```

### 4. Check Status

```bash
# Verify all pods are running
kubectl get pods -n magento

# Expected output:
# NAME                         READY   STATUS    RESTARTS   AGE
# elasticsearch-0              1/1     Running   0          5m
# magento-xxxxx-xxxxx         1/1     Running   0          3m
# magento-xxxxx-xxxxx         1/1     Running   0          3m
# mysql-0                      1/1     Running   0          6m
# redis-xxxxx-xxxxx           1/1     Running   0          5m
```

### 5. Access Magento

#### Option A: Via LoadBalancer (if supported)
```bash
kubectl get svc magento -n magento
# Look for EXTERNAL-IP and visit http://<EXTERNAL-IP>
```

#### Option B: Via Port Forward
```bash
kubectl port-forward -n magento svc/magento 8080:80
# Open http://localhost:8080 in your browser
```

## What's Next?

- [Complete Deployment Guide](DEPLOYMENT.md) - Detailed instructions
- [Production Checklist](PRODUCTION-CHECKLIST.md) - Make it production-ready
- [Cloud Examples](examples/) - AWS, GCP, Azure specific configurations

## Troubleshooting

### Pods not starting?
```bash
kubectl describe pod <pod-name> -n magento
kubectl logs <pod-name> -n magento
```

### Storage issues?
```bash
kubectl get pv
kubectl get pvc -n magento
```

### Need help?
Check the [Deployment Guide](DEPLOYMENT.md) for detailed troubleshooting steps.

## Cleanup

To remove everything:
```bash
kubectl delete -k .
# Or delete the namespace
kubectl delete namespace magento
```

## Architecture

```
┌─────────────────┐
│   Ingress       │  ← External traffic (HTTPS)
└────────┬────────┘
         │
┌────────▼────────┐
│   Magento       │  ← 2+ replicas (auto-scaling)
│   (Apache/PHP)  │
└─┬───┬───┬───────┘
  │   │   │
  │   │   └──────┐
  │   │          │
┌─▼──┐ ┌─▼────┐ ┌▼──────────┐
│MySQL│Redis  │Elasticsearch│
└────┘ └──────┘ └───────────┘
  │      │          │
  └──────┴──────────┘
  Persistent Storage
```

## Components

- **Magento 2**: eCommerce application
- **MySQL 8.0**: Database
- **Redis 7**: Cache & sessions
- **Elasticsearch 7.17**: Search engine
- **HPA**: Auto-scaling based on CPU/memory
- **Network Policies**: Security isolation

## Features

✅ Production-ready configuration  
✅ High availability setup  
✅ Auto-scaling enabled  
✅ Security hardened  
✅ Health checks configured  
✅ Easy to customize  

## Support

- 📚 [Full Documentation](README.md)
- 🚀 [Deployment Guide](DEPLOYMENT.md)
- ☁️ [Cloud Examples](examples/)
- 🐛 [Report Issues](https://github.com/Genaker/Magento-Kubernetes/issues)
