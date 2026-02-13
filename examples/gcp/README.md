# GCP GKE Deployment Example

This example shows how to deploy Magento on Google Kubernetes Engine (GKE).

## Prerequisites

- GKE cluster
- kubectl configured for GKE
- gcloud CLI installed

## Storage Configuration

Use Google Persistent Disk:

```yaml
# gcp-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: magento-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: magento-standard
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## PersistentVolumeClaims for GCP

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: magento
spec:
  storageClassName: magento-ssd  # SSD for database
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
  storageClassName: magento-standard
  accessModes:
    - ReadWriteOnce
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
  storageClassName: magento-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

## Using Cloud SQL for MySQL

Use managed Cloud SQL instead of in-cluster MySQL:

1. Create Cloud SQL MySQL 8.0 instance
2. Enable Cloud SQL Proxy
3. Update ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: magento-config
  namespace: magento
data:
  MYSQL_HOST: "127.0.0.1"  # Via Cloud SQL Proxy sidecar
  MYSQL_PORT: "3306"
```

4. Add Cloud SQL Proxy sidecar to Magento deployment:

```yaml
containers:
  - name: cloud-sql-proxy
    image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:latest
    args:
      - "--structured-logs"
      - "--port=3306"
      - "PROJECT:REGION:INSTANCE"
    securityContext:
      runAsNonRoot: true
```

## Using Memorystore for Redis

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: magento-config
  namespace: magento
data:
  REDIS_HOST: "your-memorystore-ip"
  REDIS_PORT: "6379"
```

## LoadBalancer Service

GKE automatically provisions Google Cloud Load Balancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: magento
  namespace: magento
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
    cloud.google.com/backend-config: '{"default": "magento-backendconfig"}'
spec:
  type: LoadBalancer
  # ... rest of spec
```

## Ingress with GCE

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: magento-ingress
  namespace: magento
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "magento-ip"
    networking.gke.io/managed-certificates: "magento-cert"
    kubernetes.io/ingress.allow-http: "false"
spec:
  rules:
    - host: magento.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: magento
                port:
                  number: 80
---
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: magento-cert
  namespace: magento
spec:
  domains:
    - magento.example.com
```

## BackendConfig for Health Checks

```yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: magento-backendconfig
  namespace: magento
spec:
  healthCheck:
    checkIntervalSec: 10
    timeoutSec: 5
    healthyThreshold: 2
    unhealthyThreshold: 3
    type: HTTP
    requestPath: /health_check.php
    port: 80
  timeoutSec: 300
  connectionDraining:
    drainingTimeoutSec: 60
```

## Filestore for Shared Storage

For shared media files:

1. Create Filestore instance
2. Create PV pointing to Filestore:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: magento-filestore
spec:
  capacity:
    storage: 1Ti
  accessModes:
    - ReadWriteMany
  nfs:
    server: your-filestore-ip
    path: /magento
  mountOptions:
    - hard
    - nfsvers=3
```

## Auto-Scaling

GKE Cluster Autoscaler is enabled by default. For workload autoscaling:

```bash
# Enable Vertical Pod Autoscaler
gcloud container clusters update CLUSTER_NAME \
    --enable-vertical-pod-autoscaling

# HPA is already configured in hpa.yaml
```

## Deployment Commands

```bash
# Create GKE cluster
gcloud container clusters create magento-cluster \
  --num-nodes=3 \
  --machine-type=n1-standard-4 \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=10 \
  --enable-stackdriver-kubernetes \
  --zone=us-central1-a

# Get credentials
gcloud container clusters get-credentials magento-cluster --zone=us-central1-a

# Apply storage class
kubectl apply -f gcp-storage-class.yaml

# Deploy application
kubectl apply -k .

# Reserve static IP
gcloud compute addresses create magento-ip --global

# Get the IP
gcloud compute addresses describe magento-ip --global
```

## Monitoring with Cloud Monitoring

GKE automatically sends metrics to Cloud Monitoring:

```bash
# View logs
gcloud logging read "resource.type=k8s_container AND resource.labels.namespace_name=magento" --limit 50

# Create log-based metric
gcloud logging metrics create magento_errors \
  --description="Magento error count" \
  --log-filter='resource.type="k8s_container"
    resource.labels.namespace_name="magento"
    severity>=ERROR'
```

## Backup Strategy

1. Use Persistent Disk snapshots:
```bash
gcloud compute disks snapshot DISK_NAME --snapshot-names=magento-backup-$(date +%Y%m%d)
```

2. Use Cloud SQL automated backups
3. Store files in Cloud Storage:
```bash
kubectl exec -n magento deployment/magento -- tar czf - /var/www/html | gsutil cp - gs://your-backup-bucket/magento-$(date +%Y%m%d).tar.gz
```

## Cost Optimization

1. Use Preemptible VMs for non-critical workloads
2. Use Committed Use Discounts
3. Enable Cluster Autoscaler
4. Use appropriate machine types
5. Set up billing alerts

## Security Best Practices

1. Use Workload Identity
2. Enable Binary Authorization
3. Use Secret Manager
4. Enable VPC-native cluster
5. Use Private GKE clusters
6. Enable Shielded GKE Nodes

## Workload Identity Setup

```bash
# Enable Workload Identity on cluster
gcloud container clusters update CLUSTER_NAME \
    --workload-pool=PROJECT_ID.svc.id.goog

# Create service account
gcloud iam service-accounts create magento-sa

# Bind Kubernetes SA to Google SA
gcloud iam service-accounts add-iam-policy-binding \
    magento-sa@PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[magento/magento]"

# Annotate Kubernetes service account
kubectl annotate serviceaccount magento \
    -n magento \
    iam.gke.io/gcp-service-account=magento-sa@PROJECT_ID.iam.gserviceaccount.com
```

## Multi-Region Deployment

For high availability across regions:

1. Create GKE clusters in multiple regions
2. Use Global Load Balancer
3. Use Multi-Regional Cloud Storage
4. Use Cloud SQL with cross-region replicas
