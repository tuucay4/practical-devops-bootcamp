# Week 6 — Deploy Application on AWS EKS
Deploy a three-tier eCommerce application on AWS EKS using Kubernetes,
with persistent storage, ConfigMaps, Secrets, and ALB Ingress.

## Architecture
```
Internet
    ↓
Application Load Balancer (ALB)
    ↓ / → frontend-service:80
    ↓ /api/ → backend-service:5000
┌─────────────────────────────────────┐
│         EKS Cluster                 │
│  Namespace: ecommerce               │
│                                     │
│  ┌─────────────┐  ┌──────────────┐  │
│  │  Frontend   │  │   Backend    │  │
│  │  (2 pods)   │  │   (2 pods)   │  │
│  └─────────────┘  └──────┬───────┘  │
│                          │          │
│              ┌───────────▼───────┐  │
│              │    PostgreSQL     │  │
│              │     (1 pod)       │  │
│              │  + EBS PVC (5GB)  │  │
│              └───────────────────┘  │
└─────────────────────────────────────┘
```

## Tech Stack
- Orchestration: AWS EKS (Kubernetes 1.35)
- Worker Nodes: 2 t3.medium EC2 
- Storage: AWS EBS (5GB PersistentVolumeClaim)
- Ingress: AWS ALB

## Kubernetes Objects Created
| Object | Kind | Purpose |
|---|---|---|
| `ecommerce` | Namespace | Isolates all app resources |
| `ecommerce-config` | ConfigMap | Non-sensitive config (DB host, name) |
| `ecommerce-secret` | Secret | Sensitive config (DB credentials) |
| `postgres-pvc` | PersistentVolumeClaim | 5GB EBS volume for PostgreSQL data |
| `postgres-deployment` | Deployment | Runs PostgreSQL pod |
| `postgres-service` | Service (ClusterIP) | Internal DB endpoint |
| `backend-deployment` | Deployment | Runs 2 backend pods |
| `backend-service` | Service (ClusterIP) | Internal API endpoint |
| `frontend-deployment` | Deployment | Runs 2 frontend pods |
| `frontend-service` | Service (ClusterIP) | Internal frontend endpoint |
| `ecommerce-ingress` | Ingress | Routes external traffic via ALB |

## What I Did

### 1. Created IAM Roles
Created two IAM roles required before cluster creation:

**eks-cluster-role** — this allows EKS control plane to manage AWS resources:
- Policy: `AmazonEKSClusterPolicy`

**eks-nodegroup-role** — this allows worker nodes to communicate with EKS:
- Trusted entity: EC2 service
- Policies: `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`,
  `AmazonEC2ContainerRegistryReadOnly`

### 2. Created EKS Cluster
Created the `ecommerce-cluster` via AWS Console with:
- Kubernetes version 1.35
- Default VPC, all subnets selected
- Public endpoint access

### 3. Created Node Group
Added worker nodes via EKS Console:
- EKS automatically launched and joined EC2 instances to the cluster

### 4. Connected kubectl to EKS
Downloaded cluster credentials to local machine:
```bash
aws eks update-kubeconfig \
  --name ecommerce-cluster \
  --region us-east-1
```

### 5. Enabled OIDC Provider
The OIDC links Kubernetes service accounts to AWS IAM roles, allowing 
pods to assume IAM roles without hardcoded credentials:
```bash
eksctl utils associate-iam-oidc-provider \
  --cluster ecommerce-cluster \
  --approve \
  --region us-east-1
```

### 6. Installed EBS CSI Driver
Required for Kubernetes to provision EBS volumes for PostgreSQL:
```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster ecommerce-cluster \
  --region us-east-1
```
I also created IAM service account with OIDC to give the driver
permission to create EBS volumes in AWS.

### 7. Wrote Kubernetes Manifests and Deployed Application
Created 10 YAML files in `kubernetes/` directory covering all
Kubernetes objects and applied all manifests
```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f postgres-pvc.yaml
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```

### 9. Set Up ALB Ingress Controller
Installed the AWS Load Balancer Controller to provision
an ALB when an Ingress resource is created:

```bash
# Install via Helm
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=ecommerce-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=ROLE_ARN \
  --set region=us-east-1 \
  --set vpcId=VPC_ID
```

### 10. Created Ingress Resource
Defined routing rules, the ALB controller reads this and
creates an AWS ALB automatically:

```yaml
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /api/
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 5000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

## Key Concepts Learned

### Control Plane vs Worker Nodes
AWS manages the control plane (API server, etcd, scheduler,
controller manager) in EKS.
Worker nodes are the EC2 instances i manage via node groups.

### Namespaces
Logical isolation inside a cluster. All my app resources live
in the `ecommerce` namespace separate from `kube-system`.

### ConfigMaps vs Secrets
ConfigMaps store non-sensitive config read at runtime.
Secrets store sensitive data encoded in base64

### PersistentVolumeClaim (PVC)
A request for storage. Kubernetes fulfills it by creating an
EBS volume via the EBS CSI driver. Without a PersistentVolumeClaim,
all database data is lost when the pod restarts.

### Service Types
| Type | Access | Used for |
|---|---|---|
| ClusterIP | Inside cluster only | DB, backend, frontend |
| NodePort | Inside VPC/org | Dev/testing |
| LoadBalancer | External world | Direct exposure |
| Ingress | External with routing rules | Production |

### Ingress vs LoadBalancer Service
LoadBalancer creates one AWS ELB per service which is expensive.
Ingress creates one ALB for all services with path-based
routing, this is cost effective and more flexible.

### OIDC and IAM Roles for Service Accounts
Pods need AWS permissions to create ELBs, EBS volumes etc.
OIDC links a Kubernetes ServiceAccount to an IAM Role.
Pod assumes the role automatically with no hardcoded credentials.

### Auto-Healing
Kubernetes automatically restarts failed pods.
Tested by deleting a pod manually:
```bash
kubectl delete pod -n ecommerce -l app=backend
# Pod was immediately recreated by ReplicaSet
```

## Useful kubectl Commands
```bash
# Get all resources in ecommerce namespace
kubectl get all -n ecommerce

# Watch pods in real time
kubectl get pods -n ecommerce -w

# Check logs
kubectl logs -n ecommerce deployment/backend-deployment

# Describe a resource
kubectl describe pod -n ecommerce POD_NAME

# Restart deployment
kubectl rollout restart deployment/backend-deployment -n ecommerce
```

## Errors Encountered & Fixed
| Error | Cause | Fix |
|---|---|---|
| PVC was stuck in Pending | EBS CSI driver not installed | Installed aws-ebs-csi-driver addon |
| EBS CSI CrashLoopBackOff | Service account had no IAM role | Created iamserviceaccount with OIDC |
| PostgreSQL won't initialize | EBS volume not empty (lost+found) | Added `subPath: pgdata` to volumeMount |
| Backend ECONNREFUSED | Backend started before PostgreSQL ready | Deleted pods to force restart after DB was ready |
| ALB controller 403 DescribeRouteTables | IAM policy missing EC2 permissions | Added inline policy with required EC2 permissions |
| Ingress ADDRESS empty | Wrong IAM role being used by controller | Attached correct policy to actual role being assumed |
