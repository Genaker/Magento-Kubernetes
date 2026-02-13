# Magento Kubernetes Deployment Guide

This guide provides step-by-step instructions for deploying Magento 2 on Kubernetes.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Detailed Deployment](#detailed-deployment)
4. [Configuration](#configuration)
5. [Verification](#verification)
6. [Troubleshooting](#troubleshooting)

## Prerequisites

### Required
- Kubernetes cluster (v1.23 or higher)
- kubectl CLI tool installed and configured
- Minimum 8GB RAM and 4 CPU cores available in the cluster
- Storage provisioner (for persistent volumes)

### Optional
- Ingress controller (nginx-ingress recommended)
- cert-manager (for automatic TLS certificate management)
- Helm 3.x (if you prefer Helm deployment)
- Metrics server (for HPA to work)

### Verify Prerequisites

```bash
# Check kubectl is installed and working
kubectl version --short

# Check cluster nodes
kubectl get nodes

# Check available resources
kubectl top nodes
```

## Quick Start

Deploy everything with a single command:

```bash
# Clone the repository
git clone https://github.com/Genaker/Magento-Kubernetes.git
cd Magento-Kubernetes

# Deploy using Kustomize
kubectl apply -k .

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod --all -n magento --timeout=600s

# Get the Magento service URL
kubectl get svc magento -n magento
```

## Detailed Deployment

### Step 1: Create Namespace

```bash
kubectl apply -f namespace.yaml
```

Verify:
```bash
kubectl get namespace magento
```

### Step 2: Create Persistent Volumes

```bash
kubectl apply -f local-volumes.yaml
```

Verify:
```bash
kubectl get pv
kubectl get pvc -n magento
```

**Note**: For production, edit `local-volumes.yaml` to use your cloud provider's storage class (e.g., `gp3` for AWS, `standard-rwo` for GCP).

### Step 3: Deploy MySQL Database

```bash
kubectl apply -f mysql-deployment.yaml
```

Wait for MySQL to be ready:
```bash
kubectl wait --for=condition=ready pod -l app=mysql -n magento --timeout=300s
```

Verify:
```bash
kubectl get pods -n magento -l app=mysql
kubectl logs -n magento -l app=mysql --tail=20
```

### Step 4: Deploy Redis Cache

```bash
kubectl apply -f redis-deployment.yaml
```

Wait for Redis to be ready:
```bash
kubectl wait --for=condition=ready pod -l app=redis -n magento --timeout=180s
```

Verify:
```bash
kubectl get pods -n magento -l app=redis
kubectl exec -it -n magento deploy/redis -- redis-cli ping
```

### Step 5: Deploy Elasticsearch

```bash
kubectl apply -f elasticsearch-deployment.yaml
```

Wait for Elasticsearch to be ready:
```bash
kubectl wait --for=condition=ready pod -l app=elasticsearch -n magento --timeout=300s
```

Verify:
```bash
kubectl get pods -n magento -l app=elasticsearch
kubectl exec -it -n magento elasticsearch-0 -- curl -s http://localhost:9200/_cluster/health
```

### Step 6: Deploy Magento Application

```bash
kubectl apply -f magento-deployment.yaml
```

Wait for Magento to be ready:
```bash
kubectl wait --for=condition=ready pod -l app=magento -n magento --timeout=600s
```

Verify:
```bash
kubectl get pods -n magento -l app=magento
kubectl get svc -n magento magento
```

### Step 7: Deploy Horizontal Pod Autoscaler (Optional)

```bash
# Ensure metrics-server is installed first
kubectl apply -f hpa.yaml
```

Verify:
```bash
kubectl get hpa -n magento
```

### Step 8: Deploy Ingress (Optional)

Edit `ingress.yaml` to set your domain name:
```yaml
spec:
  tls:
    - hosts:
        - your-domain.com  # Change this
      secretName: magento-tls
  rules:
    - host: your-domain.com  # Change this
```

Then apply:
```bash
kubectl apply -f ingress.yaml
```

Verify:
```bash
kubectl get ingress -n magento
```

### Step 9: Deploy Network Policies (Optional)

For enhanced security:
```bash
kubectl apply -f network-policy.yaml
```

Verify:
```bash
kubectl get networkpolicy -n magento
```

## Configuration

### Change Database Credentials

1. Edit `mysql-deployment.yaml`
2. Change the Secret values:
```yaml
stringData:
  MYSQL_ROOT_PASSWORD: your-secure-root-password
  MYSQL_DATABASE: magento2
  MYSQL_USER: magento2
  MYSQL_PASSWORD: your-secure-password
```
3. Apply changes:
```bash
kubectl apply -f mysql-deployment.yaml
kubectl rollout restart statefulset/mysql -n magento
```

### Adjust Resource Limits

Edit the deployment files and adjust `resources` section:

```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

Apply changes:
```bash
kubectl apply -f magento-deployment.yaml
```

### Scale Magento Manually

```bash
# Scale to 5 replicas
kubectl scale deployment magento -n magento --replicas=5

# Verify scaling
kubectl get pods -n magento -l app=magento
```

### Configure Custom Domain

1. Point your domain's DNS A record to the LoadBalancer IP or Ingress IP
2. Update `ingress.yaml` with your domain
3. Apply: `kubectl apply -f ingress.yaml`

## Verification

### Check All Resources

```bash
# All pods should be Running
kubectl get pods -n magento

# All services should have endpoints
kubectl get svc -n magento
kubectl get endpoints -n magento

# Check persistent volumes are bound
kubectl get pvc -n magento
```

### Test Database Connection

```bash
# Connect to MySQL
kubectl exec -it -n magento mysql-0 -- mysql -u magento2 -p
# Enter password: magento2pass (or your custom password)

# Show databases
mysql> SHOW DATABASES;
mysql> USE magento2;
mysql> SHOW TABLES;
mysql> exit
```

### Test Redis Connection

```bash
# Test Redis
kubectl exec -it -n magento deploy/redis -- redis-cli
127.0.0.1:6379> PING
# Should return: PONG
127.0.0.1:6379> exit
```

### Test Elasticsearch

```bash
# Check Elasticsearch health
kubectl exec -it -n magento elasticsearch-0 -- curl http://localhost:9200/_cluster/health?pretty
```

### Access Magento

#### Via LoadBalancer (if supported by your cluster)
```bash
EXTERNAL_IP=$(kubectl get svc magento -n magento -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Access Magento at: http://$EXTERNAL_IP"
```

#### Via Port Forward (for testing)
```bash
kubectl port-forward -n magento svc/magento 8080:80
```
Then open http://localhost:8080 in your browser

#### Via Ingress (if configured)
Access your configured domain: https://your-domain.com

## Troubleshooting

### Pods Not Starting

```bash
# Describe the pod to see events
kubectl describe pod <pod-name> -n magento

# Check pod logs
kubectl logs <pod-name> -n magento

# Check recent events in namespace
kubectl get events -n magento --sort-by='.lastTimestamp' | tail -20
```

### Persistent Volume Issues

```bash
# Check PV status
kubectl get pv

# Check PVC status
kubectl get pvc -n magento

# Describe PVC for details
kubectl describe pvc <pvc-name> -n magento
```

### Database Connection Errors

```bash
# Check if MySQL is running
kubectl get pods -n magento -l app=mysql

# Check MySQL logs
kubectl logs -n magento mysql-0 --tail=50

# Test connection from Magento pod
kubectl exec -it -n magento <magento-pod-name> -- bash
# Inside the pod:
nc -zv mysql 3306
ping mysql
```

### Elasticsearch Not Starting

Common issue: `vm.max_map_count` too low

Solution: Set on all nodes:
```bash
# On each Kubernetes node:
sudo sysctl -w vm.max_map_count=262144

# Make it permanent:
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### LoadBalancer External IP Pending

If using a cloud provider, LoadBalancer should get an IP automatically. If on-premises:

Option 1: Use NodePort instead:
```bash
kubectl patch svc magento -n magento -p '{"spec": {"type": "NodePort"}}'
kubectl get svc magento -n magento
# Access via: http://<node-ip>:<node-port>
```

Option 2: Install MetalLB for bare-metal LoadBalancer support

### HPA Not Working

```bash
# Check if metrics-server is installed
kubectl get deployment metrics-server -n kube-system

# Install metrics-server if missing
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check HPA status
kubectl get hpa -n magento
kubectl describe hpa magento-hpa -n magento

# Check metrics are available
kubectl top pods -n magento
```

## Monitoring

### View Logs

```bash
# Tail all Magento logs
kubectl logs -n magento -l app=magento --tail=100 -f

# View logs from all containers in namespace
kubectl logs -n magento --all-containers=true --tail=100 -f

# Logs from specific pod
kubectl logs -n magento <pod-name> --tail=100 -f
```

### Check Resource Usage

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods -n magento

# Detailed pod info
kubectl describe pod <pod-name> -n magento
```

### Port Forwarding for Debugging

```bash
# Forward MySQL port
kubectl port-forward -n magento svc/mysql 3306:3306

# Forward Redis port
kubectl port-forward -n magento svc/redis 6379:6379

# Forward Elasticsearch port
kubectl port-forward -n magento svc/elasticsearch 9200:9200

# Forward Magento port
kubectl port-forward -n magento svc/magento 8080:80
```

## Cleanup

To remove all resources:

```bash
# Delete using Kustomize
kubectl delete -k .

# Or delete namespace (removes everything in it)
kubectl delete namespace magento

# Remove persistent volumes (if needed)
kubectl delete pv mysql-pv magento-pv elasticsearch-pv
```

## Next Steps

1. **Install Magento**: Connect to Magento pod and run setup
2. **Configure TLS**: Set up cert-manager for automatic HTTPS
3. **Set up backups**: Implement regular database and file backups
4. **Configure monitoring**: Install Prometheus and Grafana
5. **Optimize performance**: Tune PHP, MySQL, and Elasticsearch settings
6. **Implement CI/CD**: Set up automated deployments

## Additional Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Magento DevDocs](https://devdocs.magento.com/)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)

## Support

If you encounter issues:
1. Check the [Troubleshooting](#troubleshooting) section
2. Review pod logs and events
3. Open an issue on GitHub with detailed information
4. Include output of: `kubectl get all -n magento` and relevant pod logs
