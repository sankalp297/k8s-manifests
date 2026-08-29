# CloudDeploy — Production-Grade Microservices Platform on AWS EKS

A full end-to-end DevOps project demonstrating microservices deployment on AWS EKS with Terraform, Helm, ArgoCD GitOps, and Prometheus/Grafana observability.

## Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              AWS Cloud (ap-south-1)          │
                    │                                             │
                    │   ┌─────────────── VPC ────────────────┐    │
                    │   │                                     │    │
                    │   │   ┌── Private Subnets ──────────┐   │    │
                    │   │   │                              │   │    │
                    │   │   │   ┌─── EKS Cluster ──────┐   │   │    │
                    │   │   │   │                       │   │   │    │
                    │   │   │   │  ┌──────────────┐    │   │   │    │
                    │   │   │   │  │order-service  │    │   │   │    │
                    │   │   │   │  │  (FastAPI)    │    │   │   │    │
                    │   │   │   │  └──────┬───────┘    │   │   │    │
                    │   │   │   │         │ HTTP       │   │   │    │
                    │   │   │   │  ┌──────▼───────┐    │   │   │    │
                    │   │   │   │  │inventory-svc  │    │   │   │    │
                    │   │   │   │  │  (Express)    │    │   │   │    │
                    │   │   │   │  └──────────────┘    │   │   │    │
                    │   │   │   │                       │   │   │    │
                    │   │   │   │  ┌──────────────┐    │   │   │    │
                    │   │   │   │  │  ArgoCD       │    │   │   │    │
                    │   │   │   │  │  Prometheus   │    │   │   │    │
                    │   │   │   │  │  Grafana      │    │   │   │    │
                    │   │   │   │  └──────────────┘    │   │   │    │
                    │   │   │   └───────────────────────┘   │   │    │
                    │   │   └──────────────────────────────┘   │    │
                    │   │                                       │    │
                    │   │   ┌── Public Subnets ─┐              │    │
                    │   │   │  NAT GW, IGW      │              │    │
                    │   │   └───────────────────┘              │    │
                    │   └─────────────────────────────────────┘    │
                    │                                              │
                    │   ┌─── ECR ──────┐                          │
                    │   │ order-service │                          │
                    │   │ inventory-svc │                          │
                    │   └──────────────┘                          │
                    └─────────────────────────────────────────────┘

        ┌──────────────── CI/CD Flow ─────────────────┐
        │                                              │
        │  Developer → GitHub → ArgoCD → EKS Cluster   │
        │              (push)   (auto-sync) (deploy)   │
        └──────────────────────────────────────────────┘
```

## Tech Stack

| Category | Technology |
|----------|-----------|
| Cloud | AWS (EKS, ECR, VPC, IAM) |
| Infrastructure as Code | Terraform |
| Container Runtime | Docker |
| Orchestration | Kubernetes (EKS v1.31) |
| Package Manager | Helm v3 |
| GitOps | ArgoCD |
| Monitoring | Prometheus + Grafana |
| Backend Services | Python FastAPI, Node.js Express |
| Metrics | Prometheus client libraries |

## Repository Structure

This project spans 4 repositories:

| Repo | Description |
|------|-------------|
| [order-service](https://github.com/sankalp297/order-service) | FastAPI microservice — order management with Prometheus metrics |
| [inventory-service](https://github.com/sankalp297/inventory-service) | Express microservice — stock management with Prometheus metrics |
| [infra-terraform](https://github.com/sankalp297/infra-terraform) | Terraform IaC — VPC, EKS cluster, ECR repos (54 AWS resources) |
| [k8s-manifests](https://github.com/sankalp297/k8s-manifests) | Helm charts + ArgoCD application definitions |

## What I Built

### Microservices (Phase 1)
- Two services communicating via REST — order-service calls inventory-service to check/reserve stock
- Production Dockerfiles with multi-stage builds, non-root users
- Prometheus metrics endpoints on both services for monitoring
- Health and readiness probes for Kubernetes lifecycle management

### Infrastructure as Code (Phase 2)
- Terraform provisioning of 54 AWS resources in one command
- VPC with public/private subnets across 2 availability zones
- EKS cluster with managed node groups
- ECR repositories with image scanning enabled
- NAT Gateway for private subnet internet access
- Proper subnet tagging for EKS auto-discovery

### Kubernetes Deep Dive (Phase 3)
- Helm charts with templated deployments for both services
- Resource requests and limits for pod scheduling
- Liveness and readiness probes (HTTP health checks)
- PodDisruptionBudget — ensures availability during node maintenance
- NetworkPolicy — restricts inter-pod traffic (zero-trust networking)
- ServiceAccount with RBAC — least-privilege access control
- Kubernetes DNS-based service discovery between microservices

### GitOps with ArgoCD (Phase 4)
- ArgoCD watches the k8s-manifests repo on GitHub
- Automated sync — push to Git triggers deployment to cluster
- Self-healing — manual kubectl changes are automatically reverted
- Prune enabled — resources removed from Git are deleted from cluster

### Observability (Phase 5)
- kube-prometheus-stack deployed via Helm (Prometheus + Grafana + Alertmanager)
- Pre-built dashboards for cluster, node, and pod-level metrics
- Custom application metrics (request count, latency) exposed via /metrics
- Node exporter for infrastructure-level monitoring

## Chaos Engineering — What I Broke and Learned

| Scenario | What Happened | Key Learning |
|----------|--------------|--------------|
| Killed a pod (`kubectl delete pod`) | K8s ReplicaSet immediately created a replacement | Self-healing works — desired state is maintained automatically |
| Node resource exhaustion | Pods stuck in Pending state | Resource requests/limits matter — t3.micro couldn't fit 4 pods |
| Missing env var | Service returned "inventory service unavailable" | K8s DNS discovery requires correct service names and namespaces |
| ArgoCD self-heal test | Manual changes reverted within seconds | GitOps ensures Git is the single source of truth |

## How to Run This Project

### Prerequisites
- AWS account with CLI configured
- Terraform, kubectl, Helm, Docker installed

### Deploy Infrastructure
```bash
cd infra-terraform
terraform init
terraform apply
aws eks update-kubeconfig --name clouddeploy-cluster --region ap-south-1
```

### Build and Push Images
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com

cd order-service
docker build -t <account-id>.dkr.ecr.ap-south-1.amazonaws.com/order-service:v1 .
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/order-service:v1

cd ../inventory-service
docker build -t <account-id>.dkr.ecr.ap-south-1.amazonaws.com/inventory-service:v1 .
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/inventory-service:v1
```

### Deploy with ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f k8s-manifests/argocd/
```

### Install Monitoring
```bash
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

### Tear Down (Important — avoid charges)
```bash
terraform destroy
```

## Cost Considerations
- EKS control plane: ~$0.10/hr
- t3.small nodes (x2): ~$0.04/hr
- NAT Gateway: ~$0.045/hr
- Total: approximately ₹300-350/day
- Always run `terraform destroy` when not working

## Key Interview Talking Points
- Why Helm over raw manifests? Templating, versioning, easy rollbacks
- Why ArgoCD? GitOps pattern — Git as single source of truth, audit trail
- Why PodDisruptionBudget? Ensures availability during voluntary disruptions
- Why NetworkPolicy? Zero-trust networking — pods can't communicate unless explicitly allowed
- Why multi-stage Docker builds? Smaller images, reduced attack surface
- Why Prometheus + Grafana? Industry standard for K8s observability, custom metrics support

## Author
**Sankalp Patil** — DevOps Engineer
- GitHub: [@sankalp297](https://github.com/sankalp297)
