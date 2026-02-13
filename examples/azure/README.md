# Azure AKS Deployment Example

This example shows how to deploy Magento on Azure Kubernetes Service (AKS).

## Prerequisites

- AKS cluster
- kubectl configured for AKS
- Azure CLI installed

## Storage Configuration

Use Azure Disk and Azure Files:

```yaml
# azure-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: magento-premium-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: magento-standard-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Standard_LRS
  kind: Managed
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: magento-files
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

## PersistentVolumeClaims for Azure

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: magento
spec:
  storageClassName: magento-premium-disk  # Premium SSD for database
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
  storageClassName: magento-files  # Azure Files for shared storage
  accessModes:
    - ReadWriteMany
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
  storageClassName: magento-premium-disk
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

## Using Azure Database for MySQL

Use managed MySQL database:

1. Create Azure Database for MySQL Flexible Server
2. Configure firewall rules
3. Update ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: magento-config
  namespace: magento
data:
  MYSQL_HOST: "your-mysql-server.mysql.database.azure.com"
  MYSQL_PORT: "3306"
```

4. Use Azure Key Vault for credentials (see below)

## Using Azure Cache for Redis

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: magento-config
  namespace: magento
data:
  REDIS_HOST: "your-redis-cache.redis.cache.windows.net"
  REDIS_PORT: "6380"  # SSL port
  REDIS_USE_SSL: "true"
```

## LoadBalancer Service

AKS automatically provisions Azure Load Balancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: magento
  namespace: magento
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "false"
    service.beta.kubernetes.io/azure-pip-name: "magento-pip"
spec:
  type: LoadBalancer
  # ... rest of spec
```

## Ingress with Application Gateway

Use Azure Application Gateway Ingress Controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: magento-ingress
  namespace: magento
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/connection-draining: "true"
    appgw.ingress.kubernetes.io/connection-draining-timeout: "60"
    appgw.ingress.kubernetes.io/request-timeout: "300"
spec:
  tls:
    - hosts:
        - magento.example.com
      secretName: magento-tls
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

## Azure Key Vault Integration

Use Azure Key Vault CSI driver:

1. Install CSI driver:
```bash
helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure
```

2. Create SecretProviderClass:
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-magento-secrets
  namespace: magento
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "your-managed-identity-client-id"
    keyvaultName: "your-keyvault-name"
    tenantId: "your-tenant-id"
    objects: |
      array:
        - |
          objectName: mysql-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: mysql-root-password
          objectType: secret
          objectVersion: ""
  secretObjects:
    - secretName: mysql-secret
      type: Opaque
      data:
        - objectName: mysql-password
          key: MYSQL_PASSWORD
        - objectName: mysql-root-password
          key: MYSQL_ROOT_PASSWORD
```

3. Mount in deployment:
```yaml
volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-magento-secrets"
```

## Auto-Scaling

Enable cluster autoscaler:

```bash
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10
```

## Deployment Commands

```bash
# Create AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name magento-aks \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --enable-addons monitoring \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group myResourceGroup --name magento-aks

# Apply storage class
kubectl apply -f azure-storage-class.yaml

# Deploy application
kubectl apply -k .

# Get public IP
kubectl get svc magento -n magento
```

## Monitoring with Azure Monitor

Enable Container Insights:

```bash
az aks enable-addons \
  --resource-group myResourceGroup \
  --name magento-aks \
  --addons monitoring
```

Query logs:
```kusto
ContainerLog
| where Namespace == "magento"
| where LogEntry contains "error"
| project TimeGenerated, ContainerName, LogEntry
| take 100
```

## Backup Strategy

1. Use Azure Backup for disks:
```bash
az backup protection enable-for-vm \
  --resource-group myResourceGroup \
  --vault-name myRecoveryServicesVault \
  --vm myVM \
  --policy-name DefaultPolicy
```

2. Use Azure Database for MySQL backups
3. Store files in Azure Blob Storage:
```bash
kubectl exec -n magento deployment/magento -- tar czf - /var/www/html | \
  az storage blob upload \
    --account-name mystorageaccount \
    --container-name backups \
    --name magento-$(date +%Y%m%d).tar.gz \
    --file -
```

## Cost Optimization

1. Use Azure Reserved VM Instances
2. Use Spot VMs for non-critical workloads
3. Enable cluster autoscaler
4. Use appropriate VM sizes
5. Set up cost alerts in Azure Cost Management

## Security Best Practices

1. Use Azure Active Directory Pod Identity
2. Use Azure Policy for Kubernetes
3. Enable Azure Defender for Kubernetes
4. Use Azure Key Vault for secrets
5. Enable private AKS cluster
6. Use Network Security Groups

## Azure AD Pod Identity Setup

```bash
# Install AAD Pod Identity
kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml

# Create Azure Identity
az identity create -g myResourceGroup -n magento-identity

# Create AzureIdentity
cat <<EOF | kubectl apply -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: magento-identity
  namespace: magento
spec:
  type: 0
  resourceID: /subscriptions/SUBSCRIPTION_ID/resourcegroups/RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/magento-identity
  clientID: CLIENT_ID
EOF

# Create AzureIdentityBinding
cat <<EOF | kubectl apply -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: magento-identity-binding
  namespace: magento
spec:
  azureIdentity: magento-identity
  selector: magento-pod
EOF
```

## Multi-Region Deployment

For high availability:

1. Create AKS clusters in multiple regions
2. Use Azure Front Door for global load balancing
3. Use Azure Traffic Manager
4. Use geo-replicated storage
5. Use Azure Database for MySQL with geo-replication

## Azure Policy

Apply policies for compliance:

```bash
# Enable Azure Policy add-on
az aks enable-addons \
  --addons azure-policy \
  --name magento-aks \
  --resource-group myResourceGroup

# Assign built-in policy
az policy assignment create \
  --name 'kubernetes-cluster-containers-should-only-use-allowed-images' \
  --policy '481e6e84-6f62-4f59-b6a6-4e69b8f7a11d' \
  --scope /subscriptions/SUBSCRIPTION_ID/resourceGroups/RESOURCE_GROUP
```

## Networking

Use Azure CNI for better performance:

```bash
az aks create \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/SUB_ID/resourceGroups/RG_NAME/providers/Microsoft.Network/virtualNetworks/VNET_NAME/subnets/SUBNET_NAME
```

## Performance Tips

1. Use Premium SSD for database
2. Use Azure Files Premium for shared storage
3. Enable accelerated networking on VMs
4. Use proximity placement groups
5. Optimize VM sizes based on workload
