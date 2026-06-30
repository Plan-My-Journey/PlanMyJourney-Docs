# Plan My Journey — Deployment Guide

This guide covers the end-to-end process for provisioning infrastructure, deploying the platform to Kubernetes, and promoting releases between environments. All deployments follow a GitOps model — no `kubectl apply` is run from a developer's laptop in normal operation.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [First-Time Bootstrap](#first-time-bootstrap)
  - [1. Terraform — Infrastructure Provisioning](#1-terraform--infrastructure-provisioning)
  - [2. ArgoCD Bootstrap](#2-argocd-bootstrap)
  - [3. Karpenter Bootstrap](#3-karpenter-bootstrap)
  - [4. KEDA Bootstrap](#4-keda-bootstrap)
  - [5. KGateway Bootstrap](#5-kgateway-bootstrap)
  - [6. Secrets Setup](#6-secrets-setup)
  - [7. Platform Validation](#7-platform-validation)
- [Standard Release Process](#standard-release-process)
  - [Dev Deployment (Automatic)](#dev-deployment-automatic)
  - [Prod Promotion (Manual GitOps)](#prod-promotion-manual-gitops)
- [Environment Configuration](#environment-configuration)
- [Rollback Procedure](#rollback-procedure)
- [Disaster Recovery Deployment](#disaster-recovery-deployment)

---

## Prerequisites

| Tool | Install method | Notes |
|---|---|---|
| AWS CLI | Official installer | Configure with SSO or IAM user |
| Terraform | `tfenv` recommended | Manages multiple Terraform versions |
| kubectl | `asdf` or OS package manager | Matches cluster minor version |
| Helm | Official installer | Used for bootstrap installs only |
| ArgoCD CLI | `argocd` binary | Optional — UI available at `https://argocd.invest-iq.online` |
| GitHub CLI | `gh` | Required for triggering manual workflows |

AWS credentials for humans: use IAM Identity Center (SSO) or an IAM user with sufficient permissions for Terraform. The CI/CD pipelines use GitHub OIDC — no static credentials are stored anywhere.

---

## Repository Structure

The platform is split across four repositories:

```
PlanMyJourney-App/        Application source code (services, Dockerfiles)
PlanMyJourney-Gitops/     Helm charts + ArgoCD Application manifests + values-*.yaml
PlanMyJourney-Workflows/  Reusable GitHub Actions workflows shared across repos
PlanMyjourney-Terraform/  AWS infrastructure (VPC, EKS, RDS, IAM, SQS, …)
PlanMyJourney-Docs/       This documentation
```

The Terraform and GitOps repositories are intentionally separate so that infrastructure changes and application changes have independent approval and audit trails.

---

## First-Time Bootstrap

### 1. Terraform — Infrastructure Provisioning

Clone `PlanMyjourney-Terraform` and initialise:

```bash
git clone https://github.com/Plan-My-Journey/PlanMyjourney-Terraform.git
cd PlanMyjourney-Terraform

terraform init    # downloads providers, connects to S3 state backend
terraform plan    # review all changes
terraform apply   # takes ~15 minutes on first run
```

Resources created by Terraform:

| Resource | Notes |
|---|---|
| VPC | 10.0.0.0/16, us-east-1, public + private + database subnets, 2 NAT GWs, VPC endpoints |
| EKS cluster | `ai-travel-prod`, private worker nodes, OIDC provider |
| Managed node group | Baseline nodes, gp3 EBS, KMS encrypted |
| Karpenter node IAM role | `ai-travel-prod-karpenter-node` — referenced by EC2NodeClass |
| KEDA IRSA role | `ai-travel-prod-keda-operator` — allows `sqs:GetQueueAttributes` |
| RDS PostgreSQL Multi-AZ | Private subnets, KMS CMK, 30-day backups, deletion protection ON |
| SQS queue | `ai-travel-prod-ai-jobs` with dead letter queue |
| DynamoDB | `ai-travel-prod-ai-jobs` (async job tracking) |
| Secrets Manager | One secret per service — database URL, API keys |
| Cognito user pool | Hosted UI, RS256 JWT |
| S3 | Terraform state bucket (versioned, KMS) |
| CloudTrail | Full audit logging |

Update kubeconfig after the cluster is created:

```bash
aws eks update-kubeconfig --region us-east-1 --name ai-travel-prod
kubectl get nodes   # verify nodes appear
```

### 2. ArgoCD Bootstrap

ArgoCD is the only component installed via `helm install` directly — everything else is managed by ArgoCD afterwards.

```bash
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --values PlanMyJourney-Gitops/platform/argocd/values.yaml

# Wait for ArgoCD to become ready
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s

# Apply the root Application (app-of-apps)
kubectl apply -f PlanMyJourney-Gitops/argocd-apps/root-app.yaml
```

ArgoCD then takes over and syncs all infrastructure and application components from the `PlanMyJourney-Gitops` repository.

Access the ArgoCD UI: `https://argocd.invest-iq.online`

Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 --decode
```

### 3. Karpenter Bootstrap

Karpenter requires two prerequisites before ArgoCD can install it:

1. The Karpenter CRDs (`NodePool`, `EC2NodeClass`) must be registered.
2. The Karpenter controller must have an IRSA role.

Both are managed by Terraform. After `terraform apply` completes:

```bash
# Verify the Karpenter node IAM role exists
aws iam get-role --role-name ai-travel-prod-karpenter-node

# Verify the EC2NodeClass subnet/SG discovery tags are in place
aws ec2 describe-subnets \
  --filters "Name=tag:karpenter.sh/discovery,Values=ai-travel-prod" \
  --query 'Subnets[].SubnetId'
```

ArgoCD (after the root app is applied) installs Karpenter via the Helm chart in `PlanMyJourney-Gitops/platform/karpenter/`. The NodePool and EC2NodeClass manifests are applied in the same sync wave.

Verify Karpenter is running and ready to provision nodes:

```bash
kubectl get pods -n karpenter
kubectl get nodepools
kubectl get ec2nodeclasses
```

### 4. KEDA Bootstrap

KEDA is installed via ArgoCD from `PlanMyJourney-Gitops/platform/keda/`. No manual steps are required after the root app is applied.

Verify KEDA is operational:

```bash
kubectl get pods -n keda
kubectl get scaledobjects -n prod
kubectl get triggerauthentications -n prod
```

Verify KEDA can read the SQS queue:

```bash
kubectl describe scaledobject ai-worker-scaledobject -n prod
# Look for "Successfully set metric source" in Events
```

### 5. KGateway Bootstrap

KGateway (Envoy-based Kubernetes Gateway API implementation) is managed by ArgoCD. After the root app is applied, ArgoCD installs KGateway and applies all `Gateway` and `HTTPRoute` resources.

Verify the Gateway is programmed and the NLB is provisioned:

```bash
kubectl get gateway -n kgateway-system
# STATUS should show: Programmed=True

kubectl get httproutes -n prod
# All routes should show: ResolvedRefs=True
```

Get the NLB DNS name and verify it matches Route 53:

```bash
kubectl get gateway api-gateway -n kgateway-system \
  -o jsonpath='{.status.addresses[0].value}'
```

### 6. Secrets Setup

All secrets are managed in AWS Secrets Manager. Application pods retrieve their secrets at startup via IRSA.

Terraform creates the secrets with placeholder values. Set real values after provisioning:

```bash
# Example: user-service secret
aws secretsmanager put-secret-value \
  --secret-id ai-travel-prod/user-service \
  --secret-string '{
    "DATABASE_URL": "postgresql://user:password@<rds-endpoint>:5432/user_db",
    "COGNITO_CLIENT_ID": "<client-id>",
    "COGNITO_CLIENT_SECRET": "<client-secret>",
    "COGNITO_USER_POOL_ID": "<pool-id>"
  }'

# Repeat for travel-service, ai-service, utility-service, frontend
```

Never put secrets in Helm values files, ConfigMaps, or Docker images.

### 7. Platform Validation

After all components are running, validate the full stack:

```bash
# All pods in prod namespace should be Running
kubectl get pods -n prod

# All ArgoCD applications should show Synced + Healthy
kubectl get applications -n argocd

# KEDA ScaledObjects should show READY=True
kubectl get scaledobjects -n prod

# Karpenter NodePool should show Ready
kubectl get nodepools

# Test public endpoints
curl -I https://invest-iq.online
curl -I https://api.invest-iq.online/health
```

---

## Standard Release Process

### Dev Deployment (Automatic)

Every push to `main` in `PlanMyJourney-App` triggers the CI pipeline automatically:

```
Push to main in PlanMyJourney-App
        │
        ▼
GitHub Actions workflow:
  ├── Run unit tests
  ├── SAST scan (SonarCloud)
  ├── Container scan (Trivy)
  ├── Build Docker image (tag: git SHA)
  ├── Push to ECR
  └── Update image.tag in PlanMyJourney-Gitops → values-dev.yaml
        │
        ▼
ArgoCD detects diff in values-dev.yaml
        │
        ▼
ArgoCD syncs → dev namespace updated (rolling update, no downtime)
```

No human action is required for dev deployments.

### Prod Promotion (Manual GitOps)

Prod is not automatically deployed — a human must approve the promotion by updating `values-prod.yaml` in `PlanMyJourney-Gitops`.

**Option A — Manual GitOps commit:**

```bash
# In PlanMyJourney-Gitops:
# Edit helm-charts/<service>/values-prod.yaml
# Change image.tag from old SHA to new SHA

git add helm-charts/<service>/values-prod.yaml
git commit -m "chore: promote <service> to <sha> in prod"
git push origin main

# ArgoCD detects the change and syncs within 3 minutes
```

**Option B — Workflow dispatch:**

```bash
gh workflow run promote-to-prod.yml \
  -f service=ai-service \
  -f image_tag=20cbe9b710d42565845c4673b8238a4ebc1b9a1e
```

**Verify promotion:**

```bash
# Watch ArgoCD sync
kubectl get application -n argocd <service>-prod --watch

# Confirm new pods are running the promoted image
kubectl get pods -n prod -l app.kubernetes.io/name=<service>

# Confirm rollout completed cleanly
kubectl rollout status deployment/<service> -n prod
```

---

## Environment Configuration

Each service has two Helm values files in `PlanMyJourney-Gitops/helm-charts/<service>/`:

| File | Purpose |
|---|---|
| `values-dev.yaml` | Dev-specific overrides — updated by CI (image tag, log level) |
| `values-prod.yaml` | Prod-specific overrides — updated by humans (or prod-promote workflow) |

**Key differences between environments:**

| Setting | Prod | Dev |
|---|---|---|
| `configMap.data.LOG_LEVEL` | `warning` | `debug` |
| `configMap.data.ENVIRONMENT` | `production` | `development` |
| `replicaCount` | 2 (minimum) | 1 |
| `resources.requests.cpu` | 500m | 250m |
| `podDisruptionBudget.enabled` | `true` | `false` |
| `configMap.data.BEDROCK_MAX_TOKENS` | `4096` | `2048` |

All environment-specific values live in `values-*.yaml` files, never in the base Helm chart templates.

---

## Rollback Procedure

### Application Rollback (< 2 minutes)

Rolling back an application is a GitOps operation — revert the image tag in `values-prod.yaml`:

```bash
# In PlanMyJourney-Gitops:
git log --oneline helm-charts/<service>/values-prod.yaml   # find previous good tag
git revert HEAD                                             # or manually edit the tag
git push origin main

# ArgoCD syncs and Kubernetes performs a rolling update to the previous image
```

Kubernetes keeps the previous ReplicaSet ready so the rollback transition is seamless.

### Infrastructure Rollback (Terraform)

```bash
# Review Terraform state history in S3
aws s3api list-object-versions \
  --bucket ai-travel-prod-terraform-state \
  --prefix terraform.tfstate

# Restore a specific version, then re-plan and apply
terraform plan
terraform apply
```

### Database Rollback

RDS Multi-AZ failover is fully automatic and requires no manual steps. If a schema migration must be reverted:

```bash
# Run Alembic downgrade inside the service pod
kubectl exec -it deploy/<service> -n prod -- alembic downgrade -1
```

---

## Disaster Recovery Deployment

If the entire cluster is lost (catastrophic failure), the following steps restore full service from Git and AWS backups. Expected total recovery time is under 30 minutes.

```bash
# 1. Re-apply Terraform (re-creates VPC, EKS, RDS, SQS, IAM roles)
cd PlanMyjourney-Terraform
terraform apply

# 2. Update kubeconfig for the new cluster
aws eks update-kubeconfig --region us-east-1 --name ai-travel-prod

# 3. Bootstrap ArgoCD
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd \
  --values PlanMyJourney-Gitops/platform/argocd/values.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s

# 4. Apply root Application — ArgoCD installs everything from GitOps repo
kubectl apply -f PlanMyJourney-Gitops/argocd-apps/root-app.yaml

# 5. Monitor recovery — all apps should reach Synced + Healthy
kubectl get applications -n argocd --watch

# 6. If RDS was also lost, restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier ai-travel-prod-postgres \
  --db-snapshot-identifier <most-recent-snapshot-id> \
  --db-subnet-group-name ai-travel-prod-db-subnet \
  --vpc-security-group-ids <rds-sg-id> \
  --multi-az
```

**Expected recovery timeline:**

| Phase | Duration |
|---|---|
| Terraform apply (new cluster) | 12–15 minutes |
| ArgoCD ready | 3–5 minutes |
| All services synced and Running | 5–7 minutes |
| **Total (cluster loss)** | **< 30 minutes** |
| **RDS-only failure** | **< 2 minutes (automatic)** |

For RDS-only failure, Multi-AZ failover is fully automatic with no manual steps required.

See [ARCHITECTURE.md — Disaster Recovery Architecture](ARCHITECTURE.md#disaster-recovery-architecture) for the full failure scenario matrix and RTO/RPO targets.
