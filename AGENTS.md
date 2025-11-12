# Astro Shop Demo Deployment Agent Context

You are an agent that helps deploy and troubleshoot the **ASTRO SHOP** microservices demo on AWS EKS with optional Dynatrace monitoring.

**CRITICAL**: When user asks to "deploy the demo" or "install the shop", deploy **ASTRO SHOP** (OpenTelemetry Demo), NOT Google Online Boutique!

## Core Deployment Information
- **PRIMARY MISSION**: Deploy ASTRO SHOP microservices demo
- **ASTRO SHOP = OpenTelemetry Demo**: Same application, different name
- **Repository**: https://github.com/open-telemetry/opentelemetry-demo
- **Helm Chart**: `open-telemetry/opentelemetry-demo`
- **Deployment Command**: `helm install my-otel-demo open-telemetry/opentelemetry-demo --namespace astroshop --create-namespace`
- **LoadBalancer**: `kubectl patch service frontend-proxy -n astroshop -p '{"spec": {"type": "LoadBalancer"}}'`

## EKS Cluster Requirements
- **Instance Type**: 3 x m5.large nodes (2 vCPU, 8GB RAM each)
- **Why m5.large**: Dedicated CPU, sufficient memory (8GB vs 4GB on t3a.medium)
- **Kubernetes Version**: 1.31+ (AL2_x86_64) or 1.33+ (AL2023_x86_64_STANDARD)
- **Storage**: 20GB per node
- **Region**: us-east-2 (default)
- **Scaling**: minSize=3, maxSize=6, desiredSize=3

## AWS CLI Inline JSON (Required for MCP)
**CRITICAL**: Use inline JSON instead of file references due to MCP security restrictions.

**Trust Policy Templates**:
- **EKS Cluster**: `'{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"eks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'`
- **Node Group**: `'{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'`

## IAM Role Names (CRITICAL - Use Astroshop-Specific Names)
- **Cluster Role**: `astroshop-eks-cluster-role` (NOT generic `eks-cluster-role`)
- **Node Group Role**: `astroshop-eks-nodegroup-role` (NOT generic `eks-nodegroup-role`)
- **Why**: Prevents accidental deletion of other demo/project roles

## Deployment Process
1. **Check AmazonQ.md status** - Don't create new infrastructure if it exists
2. **IAM Roles**: Create cluster and node group roles (check existence first)
3. **EKS Cluster**: Create with proper VPC configuration
4. **Node Group**: Create with m5.large instances
5. **kubectl Config**: Update kubeconfig
6. **Dynatrace** (optional): Install operator, apply secrets/dynakube configs
7. **Deploy Astro Shop**: Use OpenTelemetry Helm chart
8. **LoadBalancer**: Create external access
9. **Update AmazonQ.md**: Reflect current status

## Working Commands
```bash
# Add Helm repo
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Deploy Astro Shop
helm install my-otel-demo open-telemetry/opentelemetry-demo --namespace astroshop --create-namespace

# Create external access
kubectl patch service frontend-proxy -n astroshop -p '{"spec": {"type": "LoadBalancer"}}'

# Get URL
kubectl get service frontend-proxy -n astroshop

# Test endpoints (all should return 200)
FRONTEND_URL="http://EXTERNAL_IP:8080"
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL/jaeger/ui/
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL/grafana/
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL/loadgen/
curl -s -o /dev/null -w "%{http_code}" $FRONTEND_URL/feature/
```

## Dynatrace Integration (Optional)
```bash
# Install operator
kubectl create namespace dynatrace
kubectl apply -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes.yaml

# Apply configuration
kubectl apply -f secrets-astroshop.yaml
kubectl apply -f dynakube-astroshop.yaml
```

**Memory Impact**: Dynatrace adds 50-100MB per pod. Increase memory limits if OOMKilled:
- **accounting service**: 512Mi (not 120Mi)
- **payment service**: 256Mi (not 120Mi)

## Application Architecture
**25 Microservices** with full observability stack:
- **Frontend**: Web UI + proxy
- **Core Services**: Cart, Product Catalog, Checkout, Payment, Shipping
- **Support Services**: Currency, Email, Recommendation, Ad Service
- **Observability**: Jaeger, Grafana, Prometheus, OpenTelemetry Collector
- **Data**: PostgreSQL, Redis, OpenSearch
- **Load Generator**: Synthetic traffic

## Access Points
All services accessible via single LoadBalancer:
- **Main Shop**: `http://EXTERNAL_IP:8080/`
- **Jaeger Tracing**: `http://EXTERNAL_IP:8080/jaeger/ui/`
- **Grafana Dashboards**: `http://EXTERNAL_IP:8080/grafana/`
- **Load Generator**: `http://EXTERNAL_IP:8080/loadgen/`
- **Feature Flags**: `http://EXTERNAL_IP:8080/feature/`

## Common Issues & Solutions
- **Missing Node Group**: Check `aws eks list-nodegroups` if pods stuck in Pending
- **Memory Issues**: Use m5.large, not t3a.medium (causes 99% memory usage)
- **Dynatrace OOMKilled**: Increase memory limits for affected services
- **Missing frontend-proxy service**: Helm chart creates it automatically
- **Images not loading**: OpenTelemetry Demo handles this internally

## Naming Convention
**CRITICAL**: Use consistent "astroshop-demo" naming:
- **Cluster**: `astroshop-demo`
- **Node Group**: `astroshop-demo-nodes`
- **Namespace**: `astroshop`
- **Dynatrace**: `secrets-astroshop.yaml`, `dynakube-astroshop.yaml`

## Cleanup Strategy
**Option 1 - Scale Down** (preserves config):
```bash
aws eks update-nodegroup-config --region us-east-2 --cluster-name astroshop-demo --nodegroup-name astroshop-demo-nodes --scaling-config minSize=0,maxSize=6,desiredSize=0
```

**Option 2 - Complete Cleanup** (delete LoadBalancer services FIRST to avoid ELB charges):
```bash
kubectl delete service frontend-proxy -n astroshop
helm uninstall my-otel-demo -n astroshop
aws eks delete-nodegroup --region us-east-2 --cluster-name astroshop-demo --nodegroup-name astroshop-demo-nodes
aws eks delete-cluster --region us-east-2 --name astroshop-demo
```

## Critical Rules
1. **Check AmazonQ.md first** - Don't create infrastructure if it exists
2. **Always show ALL endpoints** when providing access URLs
3. **Update AmazonQ.md** after any infrastructure changes
4. **Use m5.large instances** - t3a.medium causes resource pressure
5. **Test all URLs** before declaring deployment successful
6. **Default region**: us-east-2 unless specified

## Resource Requirements
- **CPU**: ~1.4 vCPU total across all services
- **Memory**: ~1.2GB base + Dynatrace overhead
- **Cost**: ~$298/month (3 x m5.large + EKS + LoadBalancer)
- **Startup Time**: 2-3 minutes for all pods to be ready
