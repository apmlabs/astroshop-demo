# Amazon Q Context - Astro Shop Demo Status

## Current Deployment Status: NO DEPLOYMENT 🚫

**Last Updated**: November 12, 2025 04:11 UTC

## Infrastructure Status
- **No active infrastructure** - all resources terminated
- **Clean slate** - ready for new deployment

## Destruction Summary
- ✅ LoadBalancer service deleted (no ELB charges)
- ✅ Astro Shop application uninstalled
- ✅ Namespace deleted
- ✅ Node group deleted
- ✅ EKS cluster deleted
- ✅ IAM roles cleaned up

## Key Context for Conversations
- **Infrastructure needed** - no existing deployment
- **Fresh deployment required** - follow full setup process
- **No preserved configuration** - start from scratch

## Available Actions
- Deploy new Astro Shop infrastructure
- Follow complete setup guide
- Create new EKS cluster with proper configuration

## Management Commands
```bash
# Check pod status
kubectl get pods --all-namespaces

# Get frontend URL
kubectl get service frontend-proxy -n astroshop

# View logs
kubectl logs -f deployment/frontend -n astroshop

# Scale services
kubectl scale deployment frontend --replicas=3 -n astroshop
```

---

## Status Templates for Different States

### When Cluster is SCALED DOWN (use this template):
```
## Current Deployment Status: SCALED DOWN 🔄

**Last Updated**: [TIMESTAMP]

## Scaled Infrastructure
- **EKS Cluster**: [CLUSTER_NAME] (us-east-2) - ACTIVE
- **Node Group**: [CLUSTER_NAME]-nodes - SCALED TO 0
- **IAM Roles**: eks-cluster-role, eks-nodegroup-role (preserved)

## Application Status
🔄 **Astro Shop Demo SCALED DOWN** (all configuration preserved)
- Cluster scaled to 0 nodes to save costs
- All Kubernetes configuration and manifests intact
- Ready for quick scale-up (5-10 minutes for full startup)
- Dynatrace operator configuration preserved

## Key Context for Conversations
- **DO NOT create new infrastructure** - existing cluster just needs scaling up
- **All configuration preserved** - no redeployment needed
- **Quick scale-up available** - just scale the node group back up
- **Cluster endpoint unchanged** - same kubeconfig works

## Available Actions
- **Scale up existing cluster** (fastest option)
- Check cluster status
- **Terminate completely** (permanent cleanup)

## Scale-Up Commands
```bash
# Scale node group back up
aws eks update-nodegroup-config --region us-east-2 --cluster-name [CLUSTER_NAME] --nodegroup-name [CLUSTER_NAME]-nodes --scaling-config minSize=3,maxSize=6,desiredSize=3

# Update kubeconfig
aws eks update-kubeconfig --region us-east-2 --name [CLUSTER_NAME]

# Check pod status
kubectl get pods --all-namespaces
```
```

### When Cluster is RUNNING (use this template):
```
## Current Deployment Status: RUNNING 🟢

**Last Updated**: [TIMESTAMP]

## Active Infrastructure
- **EKS Cluster**: [CLUSTER_NAME] (us-east-2) - ACTIVE
- **Node Group**: [CLUSTER_NAME]-nodes (3 x t3a.medium) - ACTIVE
- **IAM Roles**: eks-cluster-role, eks-nodegroup-role

## Application Status
🟢 **Astro Shop Demo RUNNING**
- All 16 microservices deployed and running
- Dynatrace monitoring active
- Load generator creating synthetic traffic
- Frontend accessible via LoadBalancer

## Access Information
- **Frontend URL**: http://[EXTERNAL_IP]:80
- **All Services**: `kubectl get services`

## Key Context for Conversations
- **Infrastructure ready** - all services operational
- **Monitoring active** - Dynatrace collecting data
- **Demo ready** - can showcase microservices architecture

## Available Actions
- Check application status
- Access demo URLs
- Monitor service performance
- **Scale down cluster** (preserves all config for later restart)
- **Terminate completely** (permanent cleanup)

## Management Commands
```bash
# Check pod status
kubectl get pods --all-namespaces

# Get frontend URL
kubectl get service frontend-external

# View logs
kubectl logs -f deployment/frontend

# Scale services
kubectl scale deployment frontend --replicas=3
```
```

### When Infrastructure is TERMINATED (use this template):
```
## Current Deployment Status: NO DEPLOYMENT 🚫

**Last Updated**: [TIMESTAMP]

## Infrastructure Status
- **No active infrastructure** - all resources terminated
- **Clean slate** - ready for new deployment

## Key Context for Conversations
- **Infrastructure needed** - no existing deployment
- **Fresh deployment required** - follow full setup process
- **No preserved configuration** - start from scratch

## Available Actions
- Deploy new Astro Shop infrastructure
- Follow complete setup guide
- Create new EKS cluster with proper configuration
```

## Infrastructure Details Template
```
- **Region**: us-east-2
- **Cluster Version**: 1.31
- **Node Type**: t3a.medium
- **Deployment Date**: [DATE]
- **Monitoring**: Dynatrace Operator + OneAgent
- **Persistence**: Kubernetes manifests and configurations
```
