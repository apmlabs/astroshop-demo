# Online Shop Demo Deployment Agent Context

You are an agent that helps deploy and troubleshoot the Online Shop microservices demo on AWS EKS with Dynatrace monitoring.

Repository URL: https://github.com/GoogleCloudPlatform/microservices-demo

## Deployment Information
- Online Shop Demo is a Kubernetes-based microservices application
- Supports deployment on AWS EKS clusters
- Includes 11 microservices + Redis + load generator
- Official repository: https://github.com/GoogleCloudPlatform/microservices-demo

## EKS Cluster Requirements
- **Minimum**: 3 x t3a.medium nodes (2 vCPU, 4GB RAM each)
- **Kubernetes Version**: 1.31 or earlier for AL2_x86_64, or 1.33+ with AL2023_x86_64_STANDARD
- **Storage**: 20GB per node
- **Networking**: Public/private subnets with internet gateway access
- **IAM**: EKS cluster role and node group role with proper policies

## Deployment Strategy
- Local system is CONTROL CENTER only - deploy to remote EKS cluster
- Use AWS CLI and kubectl for cluster management
- Dynatrace operator and OneAgent for full-stack monitoring
- **CRITICAL**: Deploy Dynatrace operator BEFORE microservices for complete visibility

## Installation Process (UPDATED ORDER - Dynatrace FIRST)
1. **IAM Roles**: Create EKS cluster role and node group role (check if exists first)
2. **EKS Cluster**: Create cluster with proper VPC configuration
3. **Node Group**: Create managed node group with t3a.medium instances
4. **kubectl Config**: Update kubeconfig for cluster access
5. **Dynatrace Operator**: **INSTALL FIRST** - Deploy operator and wait for readiness
6. **Dynatrace Config**: Apply DynaKube secrets and configuration
7. **Microservices**: Deploy online shop demo manifests
8. **Verification**: Check pod status and service accessibility

## Dynatrace Integration
- **Operator**: Kubernetes operator for OneAgent management
- **Capabilities**: Use routing and kubernetes-monitoring (avoid debugging - requires PVCs)
- **Configuration**: DynaKube secrets and configuration from separate files
- **Installation Order**: MUST be deployed BEFORE microservices containers
- **Auto-discovery**: OneAgent automatically discovers and monitors all pods

## Cluster Configuration
- **Default Region**: us-east-2 unless specified
- **Default Cluster Name**: online-shop-demo-mcp (with versioning if exists)
- **Node Configuration**: 3 nodes, t3a.medium, ON_DEMAND capacity
- **Scaling**: minSize=3, maxSize=6, desiredSize=3
- **AMI Type**: AL2023_x86_64_STANDARD for Kubernetes 1.33+

## Application Architecture
- **Frontend**: Web UI service with external load balancer
- **Cart Service**: Shopping cart management
- **Product Catalog**: Product information service
- **Checkout**: Order processing service
- **Payment**: Payment processing service
- **Shipping**: Shipping management service
- **Email Service**: Notification service
- **Currency Service**: Currency conversion
- **Recommendation**: Product recommendation engine
- **Ad Service**: Advertisement service
- **Redis**: Session and cart storage
- **Load Generator**: Synthetic traffic generation

## Common Issues & Solutions
- **ActiveGate pending**: Remove debugging capability from DynaKube (requires PVCs)
- **Nodes not ready**: Wait 3-5 minutes for node group initialization
- **Pods stuck in Init**: Wait for Dynatrace OneAgent to initialize on nodes
- **Memory issues**: Ensure t3a.medium nodes have sufficient resources
- **Network access**: Verify security groups allow required traffic
- **kubectl access**: Ensure kubeconfig is properly configured

## Useful Commands
```bash
# Check cluster status
aws eks describe-cluster --region us-east-2 --name CLUSTER_NAME

# Check node group status
aws eks describe-nodegroup --region us-east-2 --cluster-name CLUSTER_NAME --nodegroup-name CLUSTER_NAME-nodes

# Check pod status
kubectl get pods --all-namespaces

# Check services
kubectl get services

# Check Dynatrace operator
kubectl -n dynatrace get pods

# View application logs
kubectl logs -f deployment/frontend
```

## Rules
- Always update AGENTS.md when discovering new deployment insights
- **Current status is in AmazonQ.md context** - check existing deployment before creating new infrastructure
- **ALWAYS check AWS infrastructure first** - use `aws eks list-clusters` and `describe-cluster` before assuming no deployment exists
- Use AWS CLI to verify resources before creating new ones
- Document any deployment issues and their solutions
- Test application accessibility after deployment
- **Default Infrastructure Behavior**: Check AmazonQ.md first - only create new infrastructure if none exists
- **Default Region**: Use us-east-2 unless otherwise specified
- **Status Reporting**: Current deployment status is always available in AmazonQ.md context

## GitHub Repository Management
- **GitHub Setup**: Follow GITHUB.md in this folder for repository setup instructions
- **When asked about GitHub repositories**: Reference the GITHUB.md file in this project folder

## Critical LoadBalancer Cleanup Issue
**IMPORTANT**: Kubernetes LoadBalancer services create AWS ELBs that persist after cluster deletion, causing ongoing charges!

### Proper Cleanup Order (Prevents Billing Issues)
1. **Delete LoadBalancer services FIRST**: `kubectl delete service frontend-external`
2. **Then delete microservices**: `kubectl delete -f [manifests]`
3. **Then delete Dynatrace**: `kubectl delete -f [dynatrace-operator]`
4. **Finally delete infrastructure**: node group → cluster → IAM roles

**Why This Matters**: The `frontend-external` service creates an AWS ELB that continues billing even after cluster deletion if not explicitly removed first.

## Cleanup Strategy

### Option 1: Scale Down Cluster (Preserve Infrastructure)
For temporary shutdown while preserving cluster configuration:

```bash
# Scale node group to 0 (preserves cluster and configuration)
aws eks update-nodegroup-config --region us-east-2 --cluster-name CLUSTER_NAME --nodegroup-name CLUSTER_NAME-nodes --scaling-config minSize=0,maxSize=6,desiredSize=0

# Verify scaling
aws eks describe-nodegroup --region us-east-2 --cluster-name CLUSTER_NAME --nodegroup-name CLUSTER_NAME-nodes --query "nodegroup.scalingConfig"
```

**To restart later:**
```bash
# Scale node group back up
aws eks update-nodegroup-config --region us-east-2 --cluster-name CLUSTER_NAME --nodegroup-name CLUSTER_NAME-nodes --scaling-config minSize=3,maxSize=6,desiredSize=3
```

**Benefits:**
- Preserves all Kubernetes configuration
- No redeployment needed
- Faster restart than full cluster recreation
- Keeps same cluster endpoint and certificates

### Option 2: Complete Infrastructure Cleanup
When cleaning up permanently, follow this order to avoid dependency issues:

1. **Delete Dynatrace Resources**
   ```bash
   kubectl delete -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes.yaml
   ```

2. **Delete Microservices Demo**
   ```bash
   kubectl delete -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml
   ```

3. **Delete Node Group**
   ```bash
   aws eks delete-nodegroup --region us-east-2 --cluster-name CLUSTER_NAME --nodegroup-name CLUSTER_NAME-nodes
   aws eks wait nodegroup-deleted --region us-east-2 --cluster-name CLUSTER_NAME --nodegroup-name CLUSTER_NAME-nodes
   ```

4. **Delete EKS Cluster**
   ```bash
   aws eks delete-cluster --region us-east-2 --name CLUSTER_NAME
   aws eks wait cluster-deleted --region us-east-2 --name CLUSTER_NAME
   ```

5. **Clean Up IAM Roles**
   ```bash
   # EKS cluster role
   aws iam detach-role-policy --role-name eks-cluster-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
   aws iam delete-role --role-name eks-cluster-role
   
   # Node group role
   aws iam detach-role-policy --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
   aws iam detach-role-policy --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
   aws iam detach-role-policy --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
   aws iam delete-role --role-name eks-nodegroup-role
   ```

### Cleanup Verification
- Verify no running clusters: `aws eks list-clusters --region us-east-2`
- Verify no node groups: `aws eks list-nodegroups --region us-east-2 --cluster-name CLUSTER_NAME`
- Verify IAM roles cleaned: `aws iam list-roles --query "Roles[?contains(RoleName, 'eks')]"`

### Cleanup Verification
- Verify no running clusters: `aws eks list-clusters --region us-east-2`
- Verify no node groups: `aws eks list-nodegroups --region us-east-2 --cluster-name CLUSTER_NAME`
- Verify IAM roles cleaned: `aws iam list-roles --query "Roles[?contains(RoleName, 'eks')]"`
- **Critical**: Verify no running EC2 instances: `aws ec2 describe-instances --region us-east-2 --filters "Name=instance-state-name,Values=running" --query "Reservations[*].Instances[*].InstanceId"`
- **Critical**: Verify no LoadBalancers: `aws elb describe-load-balancers --region us-east-2` and `aws elbv2 describe-load-balancers --region us-east-2`

## Critical Mistakes to Avoid
- **Don't assume existing infrastructure**: Always check AWS resources first
- **Don't skip IAM role checks**: Roles may exist from previous deployments
- **Don't use hardcoded resource IDs**: VPCs, subnets vary by account/region
- **Use correct Kubernetes version**: Match AMI type with K8s version compatibility
- **Don't skip wait commands**: Use aws eks wait instead of sleep
- **Don't forget Dynatrace first**: Install operator before microservices
- **Don't use debugging capability**: Requires PVCs that may not be available
- **Always verify cluster access**: Test kubectl commands after kubeconfig update
- **Don't ignore resource limits**: Ensure nodes have sufficient CPU/memory
- **Always check pod status**: Verify all pods are running before declaring success
- **NEVER commit actual Dynatrace URLs**: Keep dynakube.yaml with [DYNATRACE_TENANT_URL] placeholder for GitHub
- **Runtime URL substitution**: Get actual URL from comment in local secrets.yaml during deployment only

## Resource Requirements Summary
- **Total CPU Requests**: ~1.4 vCPU across all microservices
- **Total Memory Requests**: ~1.2GB across all microservices
- **Dynatrace Overhead**: Additional ~200MB per node
- **Recommended**: 3 x t3a.medium nodes (6 vCPU, 12GB total)
- **Cost Estimate**: ~$95/month for compute resources
