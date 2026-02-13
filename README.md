# Magento-Kubernetes
Modern Magento 2 Kubernetes Configuration 

This repository showcases the full power of Kubernetes clusters and demonstrates how to deploy Magento 2, the world's most popular eCommerce framework, on Kubernetes, the world's most popular container orchestration platform. We provide a complete, production-ready roadmap for hosting Magento on a Kubernetes Cluster with modern best practices.

## Architecture Overview

Each component runs in a separate container or group of containers following microservices architecture:

- **Magento 2** - Frontend tier with horizontal auto-scaling
- **MySQL 8.0** - Database tier with persistent storage (StatefulSet)
- **Redis 7** - Session and cache storage
- **Elasticsearch 7.17** - Search engine and catalog indexing
- **Ingress** - External access with TLS/SSL support
- **Network Policies** - Security isolation between components

## Features

✅ **Modern Kubernetes Resources**
- StatefulSets for stateful applications (MySQL, Elasticsearch)
- Deployments for stateless applications (Magento, Redis)
- Horizontal Pod Autoscaler (HPA) for automatic scaling
- Network Policies for security isolation
- Resource requests and limits for optimal scheduling
- Health checks (liveness and readiness probes)

✅ **Production-Ready Configuration**
- Secrets management for sensitive data
- ConfigMaps for application configuration
- Persistent Volumes for data persistence
- Pod Anti-Affinity for high availability
- Rolling updates with zero downtime
- Init containers for dependency management

✅ **Security Best Practices**
- Non-root containers where possible
- Dropped capabilities and privilege escalation prevention
- Network policies for traffic isolation
- Secret-based credential management

✅ **Easy Management**
- Kustomize support for environment customization
- Organized file structure
- Comprehensive documentation

## Quick Start

### Prerequisites

- Kubernetes cluster (v1.23+)
- kubectl configured to access your cluster
- At least 8GB RAM and 4 CPU cores available
- (Optional) Ingress controller (nginx-ingress)
- (Optional) Cert-manager for TLS certificates

### Option 1: Deploy with kubectl

1. **Create the namespace and persistent volumes**
```bash
kubectl apply -f namespace.yaml
kubectl apply -f local-volumes.yaml
```

2. **Deploy the database and supporting services**
```bash
kubectl apply -f mysql-deployment.yaml
kubectl apply -f redis-deployment.yaml
kubectl apply -f elasticsearch-deployment.yaml
```

3. **Wait for services to be ready**
```bash
kubectl wait --for=condition=ready pod -l app=mysql -n magento --timeout=300s
kubectl wait --for=condition=ready pod -l app=redis -n magento --timeout=300s
kubectl wait --for=condition=ready pod -l app=elasticsearch -n magento --timeout=300s
```

4. **Deploy Magento**
```bash
kubectl apply -f magento-deployment.yaml
```

5. **Deploy autoscaling (optional but recommended)**
```bash
kubectl apply -f hpa.yaml
```

6. **Deploy Ingress for external access (optional)**
```bash
# Update ingress.yaml with your domain name first
kubectl apply -f ingress.yaml
```

7. **Deploy Network Policies for security (optional)**
```bash
kubectl apply -f network-policy.yaml
```

### Option 2: Deploy with Kustomize

```bash
# Deploy all resources at once
kubectl apply -k .

# Or build first to review
kubectl kustomize . | less
kubectl kustomize . | kubectl apply -f -
```

## Verifying the Deployment

Check the status of all pods:
```bash
kubectl get pods -n magento
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS   AGE
elasticsearch-0             1/1     Running   0          5m
magento-xxxxx-xxxxx        1/1     Running   0          3m
magento-xxxxx-xxxxx        1/1     Running   0          3m
mysql-0                     1/1     Running   0          6m
redis-xxxxx-xxxxx          1/1     Running   0          5m
```

Check services:
```bash
kubectl get svc -n magento
```

Check persistent volumes:
```bash
kubectl get pv
kubectl get pvc -n magento
```

## Accessing Magento

### Via LoadBalancer Service
```bash
kubectl get svc magento -n magento
```

Look for the EXTERNAL-IP and access Magento at `http://<EXTERNAL-IP>`

### Via Ingress
If you've configured Ingress, access Magento at your configured domain (e.g., `https://magento.example.com`)

### Via Port Forward (for testing)
```bash
kubectl port-forward -n magento svc/magento 8080:80
```
Then access at `http://localhost:8080`

## Configuration

### Customizing Database Credentials

Edit the secrets in `mysql-deployment.yaml`:
```bash
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=your-root-password \
  --from-literal=MYSQL_DATABASE=magento2 \
  --from-literal=MYSQL_USER=magento2 \
  --from-literal=MYSQL_PASSWORD=your-password \
  -n magento --dry-run=client -o yaml | kubectl apply -f -
```

### Scaling Magento

Manual scaling:
```bash
kubectl scale deployment magento -n magento --replicas=5
```

Or adjust the HPA settings in `hpa.yaml` for automatic scaling.

### Resource Allocation

Adjust resource requests and limits in deployment files based on your workload:
- `magento-deployment.yaml` - Magento application resources
- `mysql-deployment.yaml` - Database resources
- `elasticsearch-deployment.yaml` - Search engine resources
- `redis-deployment.yaml` - Cache resources

## Storage

### Local Storage (Development)

The provided configuration uses local persistent volumes suitable for development and single-node clusters.

### Production Storage

For production, use cloud provider storage classes:

**AWS EBS:**
```yaml
storageClassName: gp3
```

**Google Cloud Persistent Disk:**
```yaml
storageClassName: standard-rwo
```

**Azure Disk:**
```yaml
storageClassName: managed-premium
```

Update `local-volumes.yaml` with the appropriate `storageClassName`.

## Monitoring and Logs

View logs:
```bash
# Magento logs
kubectl logs -n magento -l app=magento --tail=100 -f

# MySQL logs
kubectl logs -n magento -l app=mysql --tail=100 -f

# All logs in namespace
kubectl logs -n magento --all-containers=true --tail=100 -f
```

Check resource usage:
```bash
kubectl top pods -n magento
kubectl top nodes
```

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod <pod-name> -n magento
kubectl get events -n magento --sort-by='.lastTimestamp'
```

### Persistent Volume issues
```bash
kubectl get pv
kubectl get pvc -n magento
kubectl describe pvc <pvc-name> -n magento
```

### Database connection issues
```bash
# Check if MySQL is ready
kubectl exec -it -n magento mysql-0 -- mysql -u root -p

# Test connectivity from Magento pod
kubectl exec -it -n magento <magento-pod> -- nc -zv mysql 3306
```

## Maintenance

### Backup

Backup persistent volumes regularly:
```bash
# Backup MySQL
kubectl exec -n magento mysql-0 -- mysqldump -u root -p<password> magento2 > magento-backup.sql

# Backup Magento files
kubectl exec -n magento <magento-pod> -- tar czf - /var/www/html > magento-files-backup.tar.gz
```

### Updates

Update Magento image:
```bash
kubectl set image deployment/magento magento=magento:2.4.6-apache -n magento
```

## Security Considerations

1. **Change default passwords** in `mysql-deployment.yaml`
2. **Use TLS/SSL** via Ingress with cert-manager
3. **Enable Network Policies** for traffic isolation
4. **Regular updates** of container images
5. **Scan images** for vulnerabilities
6. **Use RBAC** for access control
7. **Enable audit logging** in Kubernetes

## File Structure

```
.
├── namespace.yaml                 # Namespace definition
├── local-volumes.yaml            # Persistent Volumes and Claims
├── mysql-deployment.yaml         # MySQL StatefulSet and Service
├── redis-deployment.yaml         # Redis Deployment and Service
├── elasticsearch-deployment.yaml # Elasticsearch StatefulSet and Service
├── magento-deployment.yaml       # Magento Deployment and Service
├── ingress.yaml                  # Ingress for external access
├── network-policy.yaml           # Network policies for security
├── hpa.yaml                      # Horizontal Pod Autoscaler
├── kustomization.yaml            # Kustomize configuration
├── kubernetes.yml                # Legacy configuration (reference)
└── README.md                     # This file
```

## Advanced Topics

### High Availability

For HA deployment:
1. Run multiple MySQL replicas (requires MySQL replication setup)
2. Use cloud-managed databases (AWS RDS, Google Cloud SQL)
3. Implement Redis Sentinel for cache HA
4. Use Elasticsearch cluster mode with 3+ nodes
5. Deploy across multiple availability zones

### Performance Optimization

1. **Enable Varnish cache** as a separate deployment
2. **Use CDN** for static assets
3. **Optimize PHP-FPM** settings in Magento container
4. **Tune MySQL** parameters in ConfigMap
5. **Scale horizontally** using HPA
6. **Use Redis for full-page cache**

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Test your changes
4. Submit a pull request

## License

This project is open source and available under the MIT License.

## Support

For issues and questions:
- Open an issue on GitHub
- Check Magento official documentation
- Refer to Kubernetes documentation

## References

- [Magento 2 Documentation](https://devdocs.magento.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Magento Cloud Docker](https://github.com/magento/magento-cloud-docker)

---

**Note**: This configuration is meant as a starting point. Adjust resource allocations, scaling parameters, and configurations based on your specific requirements and environment.
