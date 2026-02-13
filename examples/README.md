# Cloud Provider Examples

This directory contains specific deployment examples and configurations for major cloud providers.

## Available Examples

### [AWS (Amazon Web Services)](aws/)
- EKS (Elastic Kubernetes Service) deployment
- Using AWS EBS for storage
- Integration with RDS Aurora MySQL
- Integration with ElastiCache Redis
- AWS Load Balancer Controller
- Cost optimization tips
- [Read more →](aws/README.md)

### [GCP (Google Cloud Platform)](gcp/)
- GKE (Google Kubernetes Engine) deployment
- Using Google Persistent Disk
- Integration with Cloud SQL
- Integration with Memorystore
- Google Cloud Load Balancer
- Workload Identity setup
- [Read more →](gcp/README.md)

### [Azure](azure/)
- AKS (Azure Kubernetes Service) deployment
- Using Azure Disk and Azure Files
- Integration with Azure Database for MySQL
- Integration with Azure Cache for Redis
- Application Gateway Ingress Controller
- Azure AD Pod Identity
- [Read more →](azure/README.md)

## Choosing a Cloud Provider

### AWS
**Best for:**
- Mature ecosystem with extensive services
- Strong marketplace presence
- Global reach with most regions
- Flexible pricing options

**Considerations:**
- More complex service configurations
- Steep learning curve
- VPC and networking setup required

### GCP
**Best for:**
- Kubernetes-native features (GKE is excellent)
- Simpler pricing model
- Better built-in security features
- Great for data analytics integration

**Considerations:**
- Fewer regions than AWS
- Smaller ecosystem
- Less enterprise adoption

### Azure
**Best for:**
- Microsoft stack integration
- Hybrid cloud scenarios
- Enterprise agreements
- Active Directory integration

**Considerations:**
- Can be more expensive
- Less mature Kubernetes offering
- Documentation can be complex

## Common Patterns Across Providers

### Storage

| Provider | Block Storage | Shared Storage | Managed |
|----------|--------------|----------------|---------|
| AWS      | EBS          | EFS            | Yes     |
| GCP      | Persistent Disk | Filestore   | Yes     |
| Azure    | Managed Disk | Azure Files    | Yes     |

### Managed Databases

| Provider | MySQL Service | Redis Service |
|----------|--------------|---------------|
| AWS      | RDS Aurora   | ElastiCache   |
| GCP      | Cloud SQL    | Memorystore   |
| Azure    | MySQL Flexible | Azure Cache  |

### Load Balancers

| Provider | Type | Features |
|----------|------|----------|
| AWS      | ALB/NLB | Layer 7/4, WAF integration |
| GCP      | Cloud Load Balancing | Global, integrated with CDN |
| Azure    | Application Gateway | Layer 7, WAF, SSL offload |

## Deployment Strategy

### Development Environment
- Use local storage (hostPath)
- Single replica deployments
- Lower resource limits
- No managed services (cost saving)

### Staging Environment
- Use cloud storage (with cheaper tiers)
- 2 replicas for testing HA
- Moderate resource limits
- Optional managed services

### Production Environment
- Use premium cloud storage
- 3+ replicas for HA
- Appropriate resource limits
- **Always** use managed databases
- Enable monitoring and logging
- Configure auto-scaling
- Set up disaster recovery

## Migration Between Providers

### Preparation
1. Export data from existing deployment
2. Document all configurations
3. Test backup and restore procedures
4. Plan DNS cutover

### Data Migration
```bash
# Export MySQL database
kubectl exec -n magento mysql-0 -- mysqldump -u root -p magento2 > magento.sql

# Export Magento files
kubectl exec -n magento deployment/magento -- tar czf - /var/www/html > magento-files.tar.gz

# Import to new cluster
kubectl exec -i -n magento mysql-0 -- mysql -u root -p magento2 < magento.sql
kubectl cp magento-files.tar.gz magento/deployment/magento:/tmp/
```

### DNS Cutover
1. Lower TTL values 24-48 hours before migration
2. Test new environment thoroughly
3. Update DNS records
4. Monitor both environments during transition
5. Keep old environment for 24-48 hours

## Cost Comparison

### Small Deployment (Development)
- 3 nodes, 4 vCPU, 16GB RAM each
- 500GB storage
- Standard load balancer

| Provider | Estimated Monthly Cost |
|----------|----------------------|
| AWS      | $300-400            |
| GCP      | $250-350            |
| Azure    | $350-450            |

### Medium Deployment (Production)
- 5 nodes, 8 vCPU, 32GB RAM each
- 2TB storage
- Managed database
- Managed Redis
- Premium load balancer

| Provider | Estimated Monthly Cost |
|----------|----------------------|
| AWS      | $1,500-2,000        |
| GCP      | $1,300-1,800        |
| Azure    | $1,600-2,200        |

### Large Deployment (Enterprise)
- 10+ nodes, 16 vCPU, 64GB RAM each
- 10TB storage
- Multi-region
- All managed services
- Premium support

| Provider | Estimated Monthly Cost |
|----------|----------------------|
| AWS      | $5,000-8,000        |
| GCP      | $4,500-7,000        |
| Azure    | $5,500-8,500        |

*Note: Costs are estimates and vary based on region, usage, and discounts.*

## Best Practices for All Providers

### Security
1. Use managed identity/service accounts
2. Enable encryption at rest
3. Use secrets management services
4. Enable audit logging
5. Regular security scanning

### High Availability
1. Deploy across multiple availability zones
2. Use managed services for databases
3. Configure proper health checks
4. Set up monitoring and alerting
5. Regular disaster recovery testing

### Performance
1. Use premium storage for databases
2. Enable auto-scaling
3. Use CDN for static assets
4. Optimize container images
5. Regular performance testing

### Cost Optimization
1. Use auto-scaling to match demand
2. Use spot/preemptible instances where appropriate
3. Right-size your resources
4. Use reserved instances for predictable workloads
5. Regular cost reviews

## Support Resources

### AWS
- [AWS Support](https://aws.amazon.com/premiumsupport/)
- [AWS Documentation](https://docs.aws.amazon.com/)
- [EKS Workshop](https://www.eksworkshop.com/)

### GCP
- [Google Cloud Support](https://cloud.google.com/support)
- [GCP Documentation](https://cloud.google.com/docs)
- [GKE Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)

### Azure
- [Azure Support](https://azure.microsoft.com/support/)
- [Azure Documentation](https://docs.microsoft.com/azure/)
- [AKS Best Practices](https://docs.microsoft.com/azure/aks/best-practices)

## Contributing

To add examples for additional cloud providers:
1. Create a new directory with the provider name
2. Include a detailed README.md
3. Provide example configuration files
4. Document provider-specific best practices
5. Submit a pull request
