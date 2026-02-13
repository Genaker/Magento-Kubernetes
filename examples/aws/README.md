# AWS EKS Deployment Example

This example shows how to deploy Magento on AWS EKS with AWS-specific services.

## Prerequisites

- AWS EKS cluster
- kubectl configured for EKS
- AWS Load Balancer Controller installed
- EBS CSI driver installed

## Storage Configuration

Use AWS EBS for persistent storage:

```yaml
# aws-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: magento-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: magento-io2
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## PersistentVolumeClaims for AWS

Replace the PVC definitions in `local-volumes.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: magento
spec:
  storageClassName: magento-io2  # High-performance storage for database
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: magento-pvc
  namespace: magento
spec:
  storageClassName: magento-gp3  # Standard storage for files
  accessModes:
    - ReadWriteMany  # For EFS, use ReadWriteMany
  resources:
    requests:
      storage: 200Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: elasticsearch-pvc
  namespace: magento
spec:
  storageClassName: magento-gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

## Using AWS RDS for MySQL

Instead of running MySQL in-cluster, use AWS RDS Aurora:

1. Create RDS Aurora MySQL 8.0 cluster
2. Note the endpoint and credentials
3. Update ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: magento-config
  namespace: magento
data:
  MYSQL_HOST: "your-rds-endpoint.region.rds.amazonaws.com"
  MYSQL_PORT: "3306"
  # ... rest of config
```

4. Skip deploying `mysql-deployment.yaml`

## Using AWS ElastiCache for Redis

Use managed Redis:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: magento-config
  namespace: magento
data:
  REDIS_HOST: "your-elasticache-endpoint.cache.amazonaws.com"
  REDIS_PORT: "6379"
```

Skip deploying `redis-deployment.yaml`

## LoadBalancer with AWS ALB

Update service to use AWS Load Balancer Controller:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: magento
  namespace: magento
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
spec:
  type: LoadBalancer
  # ... rest of spec
```

## Ingress with ALB

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: magento-ingress
  namespace: magento
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/xxxxx
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
    - host: magento.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: magento
                port:
                  number: 80
```

## EFS for Shared Storage

For shared media files, use EFS:

1. Create EFS filesystem
2. Install EFS CSI driver
3. Create StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxx
  directoryPerms: "700"
```

## Auto-Scaling with Cluster Autoscaler

Install AWS Cluster Autoscaler to automatically scale nodes:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

## Deployment Commands

```bash
# Apply storage class
kubectl apply -f aws-storage-class.yaml

# Deploy the application
kubectl apply -k .

# Get LoadBalancer URL
kubectl get svc magento -n magento
```

## Cost Optimization

1. Use Spot Instances for non-critical workloads
2. Right-size your instances
3. Use Savings Plans
4. Enable AWS Cost Explorer
5. Set up billing alerts

## Monitoring with CloudWatch

Enable Container Insights:

```bash
aws eks update-cluster-config \
  --name your-cluster-name \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

## Backup Strategy

1. Use AWS Backup for EBS volumes
2. Use RDS automated backups
3. Store Magento files backup in S3

```bash
# Example backup script
kubectl exec -n magento deployment/magento -- tar czf - /var/www/html | aws s3 cp - s3://your-backup-bucket/magento-$(date +%Y%m%d).tar.gz
```

## Security Best Practices

1. Use AWS Secrets Manager or Parameter Store
2. Enable VPC Flow Logs
3. Use Security Groups properly
4. Enable AWS GuardDuty
5. Regular security audits with AWS Security Hub
