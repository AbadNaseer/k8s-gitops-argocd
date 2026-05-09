# k8s-gitops-argocd

Production-grade GitOps platform using **ArgoCD** and **Helm** on Kubernetes (EKS/AKS/GKE). Implements the App-of-Apps pattern for multi-environment microservice deployments with automated sync, drift detection, and rollback.

## Architecture

```
GitHub Repo (source of truth)
        │
        ▼
  ArgoCD (App-of-Apps)
        │
   ┌────┴────┐
   ▼         ▼
dev ns    staging ns    prod ns
   │         │              │
Helm charts per microservice (api, worker, frontend)
   │
HPA + Ingress + RBAC + NetworkPolicy
```

## Stack

| Component | Tool |
|-----------|------|
| GitOps controller | ArgoCD |
| Package management | Helm |
| Cluster provisioning | Terraform (EKS) |
| CI pipeline | GitHub Actions |
| Ingress | Nginx Ingress Controller + cert-manager |
| Autoscaling | HPA (CPU + custom metrics) |
| Secrets | External Secrets Operator + AWS Secrets Manager |
| RBAC | Kubernetes RBAC + ArgoCD RBAC |

## Structure

```
k8s-gitops-argocd/
├── argocd/
│   ├── install/
│   │   └── argocd-install.yaml       # ArgoCD namespace + install
│   ├── app-of-apps.yaml              # Root app managing all child apps
│   └── apps/
│       ├── api-service-dev.yaml
│       ├── api-service-staging.yaml
│       ├── api-service-prod.yaml
│       └── worker-service-prod.yaml
├── charts/
│   ├── api-service/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── values-dev.yaml
│   │   ├── values-staging.yaml
│   │   ├── values-prod.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── hpa.yaml
│   │       ├── ingress.yaml
│   │       ├── networkpolicy.yaml
│   │       └── serviceaccount.yaml
│   └── worker-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── hpa.yaml
├── terraform/
│   ├── main.tf                       # EKS cluster + node groups
│   ├── variables.tf
│   ├── outputs.tf
│   └── modules/
│       └── eks/
│           ├── main.tf
│           ├── variables.tf
│           └── outputs.tf
├── .github/
│   └── workflows/
│       ├── helm-lint.yaml            # Lint + dry-run on PR
│       └── image-build-push.yaml    # Build → ECR → update image tag
└── README.md
```

## Quick Start

### 1. Provision EKS cluster
```bash
cd terraform/
terraform init
terraform plan -var-file="environments/prod.tfvars"
terraform apply -var-file="environments/prod.tfvars"

# Update kubeconfig
aws eks update-kubeconfig --name prod-cluster --region us-east-1
```

### 2. Install ArgoCD
```bash
kubectl apply -f argocd/install/argocd-install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=120s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### 3. Bootstrap App-of-Apps
```bash
kubectl apply -f argocd/app-of-apps.yaml
```
ArgoCD will detect and sync all child apps automatically.

### 4. Access ArgoCD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
```

## Deployment Flow

```
Developer pushes code
        │
        ▼
GitHub Actions: build image → push to ECR → update image tag in values-prod.yaml
        │
        ▼
ArgoCD detects drift (polling every 3 min or webhook)
        │
        ▼
ArgoCD syncs Helm release → rolling update with zero downtime
        │
        ▼
Health checks pass → sync marked Healthy
  (or fails → ArgoCD pauses, Slack alert fires)
```

## Environment Promotion

```bash
# Promote staging → prod by updating image tag
./scripts/promote.sh --from staging --to prod --service api-service --tag v1.4.2
```

## Key Features

- **App-of-Apps pattern** — single root app manages all child applications
- **Per-environment Helm values** — `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`
- **Automated drift detection** — ArgoCD polls every 3 min; webhook for instant sync
- **Progressive delivery** — HPA scales on CPU + custom Prometheus metrics
- **Secret management** — External Secrets Operator pulls from AWS Secrets Manager; no plaintext in Git
- **RBAC** — ArgoCD projects restrict which repos/clusters each team can deploy to
- **Network isolation** — NetworkPolicies enforce namespace-level ingress/egress rules

## Monitoring

ArgoCD sync status and health exposed via Prometheus metrics:
```
argocd_app_info{name="api-service-prod", health_status="Healthy", sync_status="Synced"}
argocd_app_sync_total{name="api-service-prod", phase="Succeeded"}
```
Grafana dashboard: `dashboards/argocd-overview.json`
