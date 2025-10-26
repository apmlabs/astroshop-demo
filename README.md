# 🚀 Astro Shop Demo on AWS EKS

A complete microservices-based e-commerce platform showcasing cloud-native architecture, observability, and modern deployment practices on Amazon EKS.

> **📊 Current Status**: Check [AmazonQ.md](./AmazonQ.md) for live deployment information

## What is Astro Shop?

Astro Shop is the **OpenTelemetry Demo Application** - a fully-featured e-commerce platform built with 25+ microservices. It demonstrates:

- **Cloud-native microservices architecture** with realistic service interactions
- **Complete e-commerce workflows** - product browsing, cart management, checkout, and payment processing
- **Full observability stack** with OpenTelemetry, Jaeger, Grafana, and Prometheus
- **Production-ready deployment** on AWS EKS with Dynatrace monitoring
- **Built-in load generation** for realistic traffic simulation

## Prerequisites

- AWS account with EKS permissions
- Basic knowledge of Kubernetes and AWS
- `kubectl`, `helm`, and AWS CLI installed

## Infrastructure Requirements

**Recommended Configuration:**
- **Nodes**: 3 x m5.large instances (2 vCPU, 8GB RAM each)
- **Kubernetes**: Version 1.31+ 
- **Storage**: 20GB per node
- **Region**: us-east-2 (configurable)

> **💡 Why m5.large?** Dedicated CPU and sufficient memory (8GB) prevent resource pressure issues that occur with smaller burstable instances.

## Architecture Overview

| Component | Purpose | Resources |
|-----------|---------|-----------|
| Frontend | Web UI and customer interface | 100m CPU, 64Mi RAM |
| Cart Service | Shopping cart management | 200m CPU, 64Mi RAM |
| Product Catalog | Product information and search | 100m CPU, 64Mi RAM |
| Checkout | Order processing | 100m CPU, 64Mi RAM |
| Payment | Payment processing | 100m CPU, 256Mi RAM* |
| Accounting | Financial tracking | 100m CPU, 512Mi RAM* |
| Shipping | Delivery management | 100m CPU, 64Mi RAM |
| Recommendation | AI-powered suggestions | 200m CPU, 220Mi RAM |
| Load Generator | Synthetic traffic | 500m CPU, 256Mi RAM |

*Higher memory allocation required for Dynatrace agent injection

## Quick Start

### 1. Create EKS Cluster

```bash
# Set up IAM roles
aws iam create-role --role-name eks-cluster-role --assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "eks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}'

aws iam attach-role-policy --role-name eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# Create cluster
CLUSTER_NAME="astroshop-demo"
aws eks create-cluster --region us-east-2 --name $CLUSTER_NAME \
  --version 1.31 \
  --role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/eks-cluster-role \
  --resources-vpc-config subnetIds=$(aws ec2 describe-subnets --region us-east-2 --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --region us-east-2 --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)" --query "Subnets[*].SubnetId" --output text | tr '\t' ',')

# Wait for cluster
aws eks wait cluster-active --region us-east-2 --name $CLUSTER_NAME
```

### 2. Create Node Group

```bash
# Create node group role
aws iam create-role --role-name eks-nodegroup-role --assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}'

# Attach required policies
aws iam attach-role-policy --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name eks-nodegroup-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# Create node group
aws eks create-nodegroup --region us-east-2 --cluster-name $CLUSTER_NAME \
  --nodegroup-name $CLUSTER_NAME-nodes \
  --node-role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/eks-nodegroup-role \
  --subnets $(aws ec2 describe-subnets --region us-east-2 --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --region us-east-2 --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)" --query "Subnets[*].SubnetId" --output text | tr '\t' ' ') \
  --instance-types m5.large \
  --scaling-config minSize=3,maxSize=6,desiredSize=3 \
  --disk-size 20 \
  --ami-type AL2023_x86_64_STANDARD

# Configure kubectl
aws eks update-kubeconfig --region us-east-2 --name $CLUSTER_NAME
```

### 3. Install Dynatrace Operator (Optional)

```bash
kubectl create namespace dynatrace
kubectl apply -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes.yaml

# Apply your Dynatrace configuration
kubectl apply -f secrets-astroshop.yaml
kubectl apply -f dynakube-astroshop.yaml
```

### 4. Deploy Astro Shop

```bash
# Add OpenTelemetry Helm repository
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Install DAPR (required dependency)
dapr init -k --wait

# Deploy Astro Shop
helm install my-otel-demo open-telemetry/opentelemetry-demo \
  --namespace astroshop --create-namespace

# Create external access
kubectl patch service frontend-proxy -n astroshop \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

### 5. Access Your Application

```bash
# Get the external URL
kubectl get service frontend-proxy -n astroshop

# Access points:
# Main Shop: http://EXTERNAL-IP:8080
# Grafana: http://EXTERNAL-IP:8080/grafana/
# Jaeger: http://EXTERNAL-IP:8080/jaeger/ui/
# Load Generator: http://EXTERNAL-IP:8080/loadgen/
# Feature Flags: http://EXTERNAL-IP:8080/feature/
```

## Management

### Monitor Application
```bash
kubectl get pods -n astroshop
kubectl logs -f deployment/frontend -n astroshop
```

### Scale Services
```bash
kubectl scale deployment frontend --replicas=3 -n astroshop
```

### Troubleshooting
```bash
# Check pod status
kubectl describe pod POD_NAME -n astroshop

# View service logs
kubectl logs deployment/SERVICE_NAME -n astroshop

# Check resource usage
kubectl top nodes
kubectl top pods -n astroshop
```

## Cost Management

**Monthly Costs (us-east-2):**
- 3 x m5.large nodes: ~$207/month
- EKS cluster: $73/month
- LoadBalancer: ~$18/month
- **Total**: ~$298/month

**Cost Optimization:**
- Scale down node group when not in use: `aws eks update-nodegroup-config --scaling-config minSize=0,desiredSize=0`
- Use Spot instances for development (add `--capacity-type SPOT`)
- Set up CloudWatch billing alerts

## Cleanup

**⚠️ Important**: Delete LoadBalancer services first to avoid ongoing ELB charges!

```bash
# Delete application
kubectl delete service frontend-proxy -n astroshop
helm uninstall my-otel-demo -n astroshop

# Delete infrastructure
aws eks delete-nodegroup --region us-east-2 --cluster-name $CLUSTER_NAME --nodegroup-name $CLUSTER_NAME-nodes
aws eks delete-cluster --region us-east-2 --name $CLUSTER_NAME

# Clean up IAM roles (if not used elsewhere)
aws iam detach-role-policy --role-name eks-cluster-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
aws iam delete-role --role-name eks-cluster-role
# ... repeat for nodegroup role
```

## Contributing

This repository demonstrates deployment of the [OpenTelemetry Demo Application](https://github.com/open-telemetry/opentelemetry-demo) on AWS EKS with observability best practices.

## License

This project follows the same license as the OpenTelemetry Demo Application.

---

**🔗 Links:**
- [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Dynatrace Operator](https://github.com/Dynatrace/dynatrace-operator)
