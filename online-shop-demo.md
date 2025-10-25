Context:

```
> Here's a comprehensive context for deploying an EKS cluster with Dynatrace monitoring and microservices demo:

## EKS + Dynatrace + Microservices Demo Deployment Context

### Prerequisites
• AWS CLI configured with appropriate permissions
• kubectl installed
• Working directory: ~/mcpprojects/online-shop-demo/
• Dynatrace tenant URL and tokens available in dynakube-secrets.yaml and dynakube.yaml (separate files)

### Agent Configuration Rules
• **Default AWS Region:** us-east-2 (override if user specifies different region in prompt)
• **Default Cluster Name:** online-shop-demo-mcp (override if user specifies different name in prompt)
• **Cluster Name Versioning:** If cluster exists, append -v1, -v2, etc. until unique name found
• **Always prefer AWS MCP tools (call_aws, use_aws) over execute_bash** for AWS-related tasks
• **Region/Name Handling:** Parse user prompt for region/cluster name preferences, use defaults if not specified
• **Use aws eks wait commands:** Use `aws eks wait cluster-active` and `aws eks wait nodegroup-active` instead of sleep
• **Check existing resources:** Always check if IAM roles exist before creating (they may already exist from previous deployments)

### Key Deployment Steps

**Note:** Replace all region and cluster name references with actual values parsed from user prompt or defaults.

1. Create EKS Service Role (check if exists first)
```
bash
# Check if role exists first
if ! aws iam get-role --region [REGION] --role-name eks-cluster-role 2>/dev/null; then
  # Create trust policy file
  cat > eks-service-role-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

  # Create role
  aws iam create-role --region [REGION] --role-name eks-cluster-role --assume-role-policy-document file://eks-service-role-trust-policy.json
fi

# Always attach policy (idempotent)
aws iam attach-role-policy --region [REGION] --role-name eks-cluster-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```


2. Create EKS Cluster
```
bash
# Get VPC and subnets
VPC_ID=$(aws ec2 describe-vpcs --region [REGION] --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
SUBNETS=$(aws ec2 describe-subnets --region [REGION] --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[*].SubnetId" --output text | tr '\t' ' ')

# Determine unique cluster name
CLUSTER_NAME="[CLUSTER_NAME]"
VERSION=1
while aws eks describe-cluster --region [REGION] --name $CLUSTER_NAME 2>/dev/null; do
  VERSION=$((VERSION + 1))
  CLUSTER_NAME="[CLUSTER_NAME]-v$VERSION"
done
echo "Using cluster name: $CLUSTER_NAME"

# Create cluster
aws eks create-cluster --region [REGION] --name $CLUSTER_NAME --version 1.31 --role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/eks-cluster-role --resources-vpc-config subnetIds=$SUBNETS,endpointPrivateAccess=true,endpointPublicAccess=true

# Wait for cluster to be active
aws eks wait cluster-active --region [REGION] --name $CLUSTER_NAME
```


3. Create Node Group Role (check if exists first)
```
bash
# Check if nodegroup role exists first
if ! aws iam get-role --region [REGION] --role-name eks-nodegroup-role 2>/dev/null; then
  # Create nodegroup trust policy
  cat > nodegroup-role-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

  # Create role
  aws iam create-role --region [REGION] --role-name eks-nodegroup-role --assume-role-policy-document file://nodegroup-role-trust-policy.json
fi

# Always attach policies (idempotent)
aws iam attach-role-policy --region [REGION] --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --region [REGION] --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --region [REGION] --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```


4. Create Node Group
```
bash
# Create nodegroup (cluster must be ACTIVE first)
aws eks create-nodegroup --region [REGION] --cluster-name [CLUSTER_NAME] --nodegroup-name [CLUSTER_NAME]-nodes --node-role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/eks-nodegroup-role --subnets $SUBNETS --instance-types t3a.medium --scaling-config minSize=3,maxSize=6,desiredSize=3 --disk-size 20 --ami-type AL2023_x86_64_STANDARD --capacity-type ON_DEMAND

# Wait for nodegroup to be active
aws eks wait nodegroup-active --region [REGION] --cluster-name [CLUSTER_NAME] --nodegroup-name [CLUSTER_NAME]-nodes
```


5. Configure kubectl
```
bash
aws eks update-kubeconfig --region [REGION] --name [CLUSTER_NAME]
```


6. Install Dynatrace Operator
```
bash
kubectl create namespace dynatrace
kubectl apply -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes.yaml
kubectl -n dynatrace wait --for=condition=ready pod --selector=app.kubernetes.io/name=dynatrace-operator --timeout=300s
```


7. Deploy Dynatrace Configuration
```
bash
# Apply secrets first (contains working base64-encoded tokens)
kubectl apply -f dynakube-secrets.yaml

# Then apply DynaKube configuration
kubectl apply -f dynakube.yaml
```


8. Deploy Microservices Demo
```
bash
kubectl apply -f https://raw.githubusercontent.com/apmlabs/microservices-demo/main/release/kubernetes-manifests.yaml
```


9. Get Application URL
```
bash
kubectl get service frontend-external
```


### Critical Configuration Notes

• **Kubernetes Version:** Use 1.31 or earlier for AL2_x86_64 AMI, or 1.33+ with AL2023_x86_64_STANDARD
• **DynaKube Capabilities:** Avoid debugging capability as it requires PVCs. Use only routing and kubernetes-monitoring
• **Node Sizing:** t3.medium (2 vCPU, 4GB) x 3 nodes provides optimal cost/performance for this workload
• **Resource Requirements:** Total ~1.4 vCPU requests, ~1.2GB memory requests for microservices + Dynatrace overhead

### Expected Results
• **Cluster:** 3-node EKS cluster in specified region (default: us-east-2)
• **Monitoring:** Full-stack Dynatrace monitoring with log analytics
• **Application:** 11 microservices + Redis + load generator
• **Access:** Public load balancer URL for the online shop frontend
• **Cost:** ~$95/month for compute resources

### Troubleshooting
• If ActiveGate is pending: Check for PVC requirements and remove debugging capability
• If nodes not ready: Wait 3-5 minutes for node group creation
• If pods stuck in Init: Wait for Dynatrace OneAgent to initialize on nodes

### Cleanup Instructions
**Complete teardown order (important - follow sequence):**
1. Delete Dynatrace operator: `kubectl delete -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes.yaml`
2. Delete microservices demo: `kubectl delete -f https://raw.githubusercontent.com/apmlabs/microservices-demo/main/release/kubernetes-manifests.yaml`
3. Delete nodegroup: `aws eks delete-nodegroup --region [REGION] --cluster-name [CLUSTER_NAME] --nodegroup-name [CLUSTER_NAME]-nodes`
4. Wait for nodegroup deletion: `aws eks wait nodegroup-deleted --region [REGION] --cluster-name [CLUSTER_NAME] --nodegroup-name [CLUSTER_NAME]-nodes`
5. Delete cluster: `aws eks delete-cluster --region [REGION] --name [CLUSTER_NAME]`
6. Wait for cluster deletion: `aws eks wait cluster-deleted --region [REGION] --name [CLUSTER_NAME]`
7. Clean up EKS service role:
   ```bash
   aws iam detach-role-policy --region [REGION] --role-name eks-cluster-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
   aws iam delete-role --region [REGION] --role-name eks-cluster-role
   ```
8. Clean up nodegroup role:
   ```bash
   aws iam detach-role-policy --region [REGION] --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
   aws iam detach-role-policy --region [REGION] --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
   aws iam detach-role-policy --region [REGION] --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
   aws iam delete-role --region [REGION] --role-name eks-nodegroup-role
   ```

This context provides the complete deployment pattern that was successfully executed.

```

