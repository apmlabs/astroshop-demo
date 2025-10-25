# Astro Shop Demo Deployment Agent Context

You are an agent that helps deploy and troubleshoot the **ASTRO SHOP** microservices demo on AWS EKS with Dynatrace monitoring.

**CRITICAL**: When user asks to "deploy the demo" or "install the shop", deploy **ASTRO SHOP**, NOT Google Online Boutique!

## Deployment Information
- **PRIMARY MISSION**: Deploy ASTRO SHOP microservices demo
- **PROVEN WORKING METHOD**: Use standard OpenTelemetry Helm chart directly
- **WORKING COMMAND**: `helm install my-otel-demo open-telemetry/opentelemetry-demo --namespace astroshop --create-namespace`
- **ASTRO SHOP = OpenTelemetry Demo**: They are the same application
- **NOT Google Online Boutique**: Do not deploy https://github.com/GoogleCloudPlatform/microservices-demo unless specifically requested
- **CRITICAL SUCCESS FACTORS**: 
  - Install Dynatrace operator first
  - Install DAPR operator second  
  - Deploy OpenTelemetry demo via Helm (simple approach works!)
  - Create LoadBalancer: `kubectl patch service frontend-proxy -n astroshop -p '{"spec": {"type": "LoadBalancer"}}'`
- **Don't overcomplicate**: GitOps repo adds complexity - use standard Helm chart

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

## Installation Process (PROVEN WORKING STEPS)
1. **IAM Roles**: Create EKS cluster role and node group role (check if exists first)
2. **EKS Cluster**: Create cluster with proper VPC configuration
3. **Node Group**: Create managed node group with t3a.medium instances
4. **kubectl Config**: Update kubeconfig for cluster access
5. **Dynatrace Operator**: Install operator first
6. **DAPR Operator**: Install DAPR runtime (dapr init -k --wait)
7. **Astro Shop Deployment**: Use standard OpenTelemetry Helm chart
8. **LoadBalancer**: Create external access
9. **Verification**: Test all URLs work (200 OK)

## EXACT WORKING COMMANDS (COPY-PASTE READY)
```bash
# 1. Prerequisites - Helm repo
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# 2. Install Dynatrace operator
kubectl create namespace dynatrace
kubectl apply -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes.yaml
kubectl -n dynatrace wait --for=condition=ready pod --selector=app.kubernetes.io/name=dynatrace-operator --timeout=300s

# 3. Install DAPR
dapr init -k --wait

# 4. Deploy Astro Shop (THE MAGIC COMMAND!)
helm install my-otel-demo open-telemetry/opentelemetry-demo --namespace astroshop --create-namespace

# 5. Create LoadBalancer for external access
kubectl patch service frontend-proxy -n astroshop -p '{"spec": {"type": "LoadBalancer"}}'

# 6. Get external URL
kubectl get service frontend-proxy -n astroshop

# 7. Test URLs (should all return 200)
FRONTEND_URL="http://EXTERNAL_IP:8080"
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL/jaeger/ui/
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL/grafana/
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL/loadgen/
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL/feature/
```

## Dynatrace Integration
- **Operator**: Kubernetes operator for OneAgent management
- **Capabilities**: Use routing and kubernetes-monitoring (avoid debugging - requires PVCs)
- **Configuration**: DynaKube secrets and configuration from separate files
- **Installation Order**: MUST be deployed BEFORE microservices containers
- **Auto-discovery**: OneAgent automatically discovers and monitors all pods

### OneAgent Control Commands (for EC2-based deployments)
- **Status check**: `sudo systemctl status oneagent`
- **Version**: `sudo /opt/dynatrace/oneagent/agent/tools/oneagentctl --version`
- **Server connection**: `sudo /opt/dynatrace/oneagent/agent/tools/oneagentctl --get-server`
- **Help**: `sudo /opt/dynatrace/oneagent/agent/tools/oneagentctl --help`
- **Container monitoring**: Look for `oneagenthelper --containerd` processes in status output

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

## Dual Deployment Capability
**IMPORTANT**: Multiple online shop deployments can run simultaneously on the same EKS cluster:

### Supported Configurations
1. **Astro Shop Demo** (default namespace) - Custom microservices demo
2. **Google Online Shop Demo** (online-shop namespace) - Official Google Cloud microservices demo
3. **Both simultaneously** - Each gets its own LoadBalancer and external IP

### Deployment Commands
```bash
# Astro Shop (default namespace) - already deployed via custom manifests
kubectl apply -f [astro-shop-manifests]

# Google Online Shop (online-shop namespace)
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml
```

### Access Points
- **Astro Shop**: `kubectl get service frontend-external` (default namespace)
- **Google Shop**: `kubectl get service online-shop-frontend -n online-shop`
- **Both create separate AWS ELBs** - important for cleanup costs

### Resource Impact
- **CPU**: ~2.8 vCPU total for dual deployment
- **Memory**: ~2.4GB total for dual deployment  
- **Nodes**: 3 x t3a.medium can handle both deployments
- **Cost**: Each LoadBalancer creates separate AWS ELB charges

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
- **CRITICAL: When user says "remember" - IMMEDIATELY update AGENTS.md with that information**
- **Current status is in AmazonQ.md context** - check existing deployment before creating new infrastructure
- **ALWAYS check AWS infrastructure first** - use `aws eks list-clusters` and `describe-cluster` before assuming no deployment exists
- Use AWS CLI to verify resources before creating new ones
- Document any deployment issues and their solutions
- Test application accessibility after deployment
- **Default Infrastructure Behavior**: Check AmazonQ.md first - only create new infrastructure if none exists
- **Default Region**: Use us-east-2 unless otherwise specified
- **Status Reporting**: Current deployment status is always available in AmazonQ.md context
- **CRITICAL: ALWAYS UPDATE STATUS FILES** - After ANY infrastructure change (start/stop/terminate/create), immediately update the online-shop-demo status documentation (AmazonQ.md) to reflect current state. Failure to update status files causes context loss and repeated mistakes across chat sessions.

## GitHub Repository Management
- **GitHub Setup**: Follow GITHUB.md in this folder for repository setup instructions
- **When asked about GitHub repositories**: Reference the GITHUB.md file in this project folder
- **Critical**: Always check .gitignore before committing - AmazonQ.md should NEVER be committed

## Critical Image Fix
**CRITICAL**: OpenTelemetry Demo has image loading issues - images don't display on website without nginx proxy fix!

### Image Problem Analysis
- Frontend service serves HTML but images fail to load
- Image-provider service runs on different port (8081) 
- Frontend requests `/images/` and `/icons/` paths 
- Image-provider serves images at ROOT level (not `/static/`)
- **Key Discovery**: Images are physically in `/static/` directory but served at root by nginx

### Root Cause
The image-provider's internal nginx serves files from `/static/` directory but exposes them at root path:
- Physical location: `/static/opentelemetry-demo-logo.png` 
- Served URL: `http://image-provider:8081/opentelemetry-demo-logo.png` (NO /static/ prefix)
- Frontend expects: `/images/opentelemetry-demo-logo.png`

### Required Fix - nginx-proxy.yaml
The nginx-proxy must route `/images/` and `/icons/` requests to `image-provider:8081/` (root, not /static/):

```yaml
location ~ ^/(images|icons)/(.*) {
    proxy_pass http://imageProvider/$2;  # Route to ROOT, not /static/$2
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

### Deployment Order for Images
1. Deploy OpenTelemetry Demo manifests
2. **Deploy nginx-proxy.yaml** (fixes images)
3. Create LoadBalancer pointing to **nginx-proxy** (not frontend)
4. **Result**: Website with working images

### Image Fix Troubleshooting Steps
**CRITICAL**: Follow these exact steps to diagnose image issues:

1. **Check image-provider has files**: 
   ```bash
   kubectl exec deployment/image-provider -- find /static -name "*.png" | head -5
   ```

2. **Test image-provider direct access** (should return 200):
   ```bash
   kubectl port-forward service/image-provider 8081:8081 &
   curl -s http://localhost:8081/opentelemetry-demo-logo.png -o /dev/null -w "%{http_code}"
   pkill -f "kubectl port-forward"
   ```

3. **Check nginx-proxy config is correct**:
   ```bash
   kubectl exec deployment/nginx-proxy -- cat /etc/nginx/nginx.conf
   # Should show: proxy_pass http://imageProvider/$2; (NOT /static/$2)
   ```

4. **Check image-provider logs for errors**:
   ```bash
   kubectl logs deployment/image-provider --tail=10
   # Look for "/static/static/" double-path errors
   ```

5. **Test final image access** (should return 200):
   ```bash
   FRONTEND_URL=$(kubectl get service frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
   curl -s http://$FRONTEND_URL/images/opentelemetry-demo-logo.png -o /dev/null -w "%{http_code}"
   ```

6. **If still broken, restart nginx-proxy**:
   ```bash
   kubectl apply -f nginx-proxy.yaml
   kubectl rollout restart deployment/nginx-proxy
   kubectl rollout status deployment/nginx-proxy
   ```

### Common Mistakes to Avoid
- **DON'T use `/static/$2`** in nginx config - images are served at root
- **DON'T assume image-provider serves at `/static/`** - it serves at root despite files being in `/static/` directory
- **ALWAYS test image-provider directly** before assuming nginx config is wrong
- **CHECK logs for double-path errors** like `/static/static/filename.png`

### Image Fix Verification
```bash
# Quick verification script
echo "Testing image fix..."
FRONTEND_URL=$(kubectl get service frontend-external -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Frontend: $(curl -s http://$FRONTEND_URL -o /dev/null -w "%{http_code}")"
echo "Images: $(curl -s http://$FRONTEND_URL/images/opentelemetry-demo-logo.png -o /dev/null -w "%{http_code}")"
echo "Icons: $(curl -s http://$FRONTEND_URL/icons/Chevron.svg -o /dev/null -w "%{http_code}")"
```

**ALWAYS deploy nginx-proxy for image fix - this is REQUIRED for functional demo!**
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
