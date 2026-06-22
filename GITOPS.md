# GitOps Operations Guide

## Overview

This project uses a **GitOps** methodology: the Git repository is the single source of truth for
the desired state of all Kubernetes resources. ArgoCD continuously reconciles the live cluster
state against what is defined in Git.

Key principles enforced:
- **No direct `kubectl apply`** for application changes — all changes flow through Git
- **Self-healing**: ArgoCD reverts manual `kubectl` changes within 3 minutes
- **Audit trail**: every change is a Git commit with author, timestamp, and diff
- **Rollback = git revert**: rolling back means reverting a commit, not running a script

---

## Repository Structure

```
ai-planner-finops/
│
├── frontend/                          # React + Vite dashboard
│   ├── src/
│   └── Dockerfile
│
├── services/                          # Python FastAPI microservices
│   ├── user-service/
│   ├── travel-service/
│   ├── ai-service/                    # AWS Bedrock integration
│   └── utility-service/
│
├── infrastructure/
│   ├── terraform/                     # Phase A — AWS infrastructure
│   │   ├── modules/
│   │   │   ├── networking/
│   │   │   ├── eks/
│   │   │   ├── rds/
│   │   │   ├── ecr/
│   │   │   ├── secrets/
│   │   │   └── finops/
│   │   ├── environments/
│   │   │   ├── prod.tfvars
│   │   │   └── prod-backend.hcl
│   │   └── main.tf
│   │
│   ├── helm-charts/                   # Phase B — Helm chart definitions
│   │   ├── frontend/
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml            # Contains image tag (updated by CI)
│   │   │   └── templates/
│   │   ├── user-service/
│   │   ├── travel-service/
│   │   ├── ai-service/
│   │   └── utility-service/
│   │
│   ├── argocd-apps/                   # Phase B — GitOps manifests
│   │   ├── app-of-apps.yaml           # Root application (manages all others)
│   │   ├── projects/
│   │   │   └── ai-travel.yaml         # ArgoCD project (permissions)
│   │   └── applications/
│   │       ├── frontend-app.yaml
│   │       ├── user-service-app.yaml
│   │       ├── travel-service-app.yaml
│   │       ├── ai-service-app.yaml
│   │       └── utility-service-app.yaml
│   │
│   └── docs/                          # This documentation
│       ├── DEPLOYMENT.md
│       ├── ARCHITECTURE.md
│       ├── GITOPS.md
│       ├── FINOPS.md
│       ├── TROUBLESHOOTING.md
│       └── SCALING.md
│
└── .github/
    └── workflows/
        ├── ci-frontend.yaml           # Frontend lint, test, build, push
        ├── ci-user-service.yaml       # User service CI
        ├── ci-travel-service.yaml
        ├── ci-ai-service.yaml
        └── ci-utility-service.yaml
```

---

## ArgoCD Concepts Used

### App-of-Apps Pattern

A single **root Application** (`app-of-apps.yaml`) is the entry point. It points to the
`infrastructure/argocd-apps/applications/` directory. ArgoCD reads that directory and
creates/manages all child Applications automatically.

```
app-of-apps (root)
    ├── Creates: frontend-app
    ├── Creates: user-service-app
    ├── Creates: travel-service-app
    ├── Creates: ai-service-app
    └── Creates: utility-service-app
```

This means deploying a new service is as simple as adding a new `.yaml` file to
`infrastructure/argocd-apps/applications/` and pushing to Git.

### Sync Waves

Sync waves control the order of resource creation within a sync operation. Resources with a
lower wave number are applied and must become healthy before higher wave resources are applied.

Annotation syntax:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

### Auto-sync vs Manual Sync

| Mode | Behavior | When to Use |
|------|---------|------------|
| Auto-sync enabled | ArgoCD applies changes automatically on git push | Normal operation (all services) |
| Auto-sync disabled | Changes require `argocd app sync <name>` command | During maintenance windows |
| Self-heal enabled | ArgoCD reverts manual `kubectl` changes | Always enabled in production |

### Pruning

When `prune: true` is set (default for all apps), ArgoCD will **delete** Kubernetes resources
that exist in the cluster but are no longer present in Git. This prevents resource drift and
zombie objects accumulating in the cluster.

---

## Deployment Workflow (End-to-End)

```
Developer                 GitHub                    ArgoCD               Kubernetes
    │                        │                          │                     │
    │──push feature branch──►│                          │                     │
    │                        │──run CI (lint/test)──────►│                    │
    │◄──CI passed────────────│                          │                     │
    │──open Pull Request─────►│                          │                     │
    │                        │──run CI (full test suite)►│                    │
    │──merge PR to main──────►│                          │                     │
    │                        │──CI: docker build──────────────────────────────►│
    │                        │──CI: docker push to ECR──────────────────────────
    │                        │──CI: update values.yaml──►│                    │
    │                        │  (image.tag: new-sha)     │                     │
    │                        │──CI: git commit [skip ci]─►│                   │
    │                        │                          │                     │
    │                        │◄─ArgoCD polls every 3min─│                     │
    │                        │                          │──detected drift──────►│
    │                        │                          │──helm upgrade────────►│
    │                        │                          │                     │──rolling update
    │                        │                          │◄─pods healthy────────│
    │                        │◄─sync complete───────────│                     │
```

### Step-by-Step

1. Developer pushes code to a feature branch (e.g., `feat/add-hotel-filter`)
2. Pull Request is opened → GitHub Actions runs: lint, unit tests, type checks
3. PR is reviewed and merged to `main`
4. On merge to `main`, GitHub Actions CI pipeline runs:
   - Builds Docker image with tag `sha-<git-short-sha>` (e.g., `sha-a1b2c3d`)
   - Pushes image to ECR
   - Updates `image.tag` in `infrastructure/helm-charts/<service>/values.yaml`
   - Commits the values change with message `[skip ci] ci: update <service> image to sha-a1b2c3d`
   - Pushes the commit back to `main`
5. ArgoCD polls the Git repository every 3 minutes (or receives webhook notification)
6. ArgoCD detects that `values.yaml` has changed (image tag differs from running pod)
7. ArgoCD runs `helm upgrade` with the new values
8. Kubernetes performs a rolling update: new pods start, old pods terminate after readiness probe passes
9. ArgoCD confirms sync complete and marks the app as `Healthy`

---

## ArgoCD CLI Cheatsheet

### Installation

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Windows (via scoop)
scoop install argocd

# Verify
argocd version
```

### Authentication

```bash
# Login with port-forward (development)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
argocd login localhost:8080 \
  --username admin \
  --password <initial-admin-password> \
  --insecure

# Login with domain (production)
argocd login argocd.aitravel.com \
  --username admin \
  --password <password>

# Logout
argocd logout argocd.aitravel.com
```

### Application Management

```bash
# List all applications and their status
argocd app list

# Get detailed status of a specific app
argocd app get frontend

# Get detailed status with tree view
argocd app get frontend --show-operation

# Sync manually (pull latest from git and apply)
argocd app sync frontend

# Sync with pruning (delete removed resources)
argocd app sync frontend --prune

# Sync only specific resources
argocd app sync frontend --resource apps:Deployment:frontend

# Force sync (ignore differences, re-apply everything)
argocd app sync frontend --force

# Watch sync status in real-time (blocks until sync complete)
argocd app wait frontend --sync --health --timeout 300

# Diff: compare live cluster state vs git
argocd app diff frontend

# Diff: compare with specific revision
argocd app diff frontend --revision HEAD~1
```

### History and Rollback

```bash
# View deployment history
argocd app history frontend
# ID  DATE                           REVISION
# 1   2026-06-01 10:00:00 +0000 UTC  abc1234
# 2   2026-06-02 11:30:00 +0000 UTC  def5678
# 3   2026-06-03 09:00:00 +0000 UTC  ghi9012

# Rollback to a previous revision (by ID from history)
argocd app rollback frontend 2

# Rollback to a specific git commit
argocd app set frontend --revision abc1234def
argocd app sync frontend

# After rollback: re-enable auto-sync (rollback disables it)
argocd app set frontend --sync-policy automated
```

### Diagnostics

```bash
# Check why an app is out of sync
argocd app get frontend --show-operation

# Get the last sync operation result
argocd app get frontend -o json | jq '.status.operationState'

# List all resources managed by an app
argocd app resources frontend

# Get logs for a resource
argocd app logs frontend --container frontend

# Check app health status
argocd app get frontend -o json | jq '.status.health'

# List apps with issues only
argocd app list | grep -v Synced
```

---

## Adding a New Microservice

Follow these steps to add a new microservice to the GitOps pipeline:

### Step 1: Create the Helm Chart

```bash
# Create chart directory
mkdir -p infrastructure/helm-charts/new-service/templates

# Create Chart.yaml
cat > infrastructure/helm-charts/new-service/Chart.yaml <<EOF
apiVersion: v2
name: new-service
description: Description of new service
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF

# Create values.yaml with defaults
cat > infrastructure/helm-charts/new-service/values.yaml <<EOF
replicaCount: 2

image:
  repository: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/ai-travel/new-service
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8000

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: true
  path: /api/new/*

serviceAccount:
  name: new-service-sa
  irsaRoleArn: ""   # Populated by Terraform output
EOF
```

### Step 2: Create Kubernetes Templates

```bash
# Add standard Deployment, Service, HPA, ServiceAccount templates
# Copy from an existing service and adjust:
cp -r infrastructure/helm-charts/user-service/templates/ \
      infrastructure/helm-charts/new-service/templates/
# Edit templates to reference {{ .Values.* }} appropriately
```

### Step 3: Create the ArgoCD Application Manifest

```bash
cat > infrastructure/argocd-apps/applications/new-service-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: new-service
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: ai-travel
  source:
    repoURL: https://github.com/your-org/ai-travel-planner.git
    targetRevision: HEAD
    path: infrastructure/helm-charts/new-service
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: ai-travel
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### Step 4: Add IRSA Role in Terraform (if AWS access needed)

```hcl
# infrastructure/terraform/modules/eks/irsa.tf
module "new_service_irsa" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  role_name = "ai-travel-new-service"
  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["ai-travel:new-service-sa"]
    }
  }
  # Add policy attachments as needed
}
```

### Step 5: Add GitHub Actions CI Workflow

```bash
cp .github/workflows/ci-user-service.yaml \
   .github/workflows/ci-new-service.yaml
# Edit: update service name, Dockerfile path, ECR repo name
```

### Step 6: Push and Verify

```bash
git add infrastructure/helm-charts/new-service/ \
        infrastructure/argocd-apps/applications/new-service-app.yaml \
        .github/workflows/ci-new-service.yaml
git commit -m "feat: add new-service to GitOps pipeline"
git push origin main

# After push: watch ArgoCD detect and sync the new app
argocd app list
argocd app wait new-service --sync --health --timeout 300
```

---

## Rollback Procedure

### Scenario 1: Rollback via ArgoCD (Recommended)

Use when: bad image was deployed and you need to quickly revert to the previous version.

```bash
# 1. Identify the last known good revision
argocd app history user-service
# Note the ID of the last working deployment

# 2. Roll back
argocd app rollback user-service <revision-id>

# 3. Monitor until healthy
argocd app wait user-service --health --timeout 120

# 4. Verify pods are running correctly
kubectl get pods -n ai-travel -l app=user-service
kubectl logs -n ai-travel -l app=user-service --tail=50
```

### Scenario 2: Git Revert (Infrastructure Changes)

Use when: a bad Terraform change or Helm chart change was merged to main.

```bash
# 1. Identify the bad commit
git log --oneline -10

# 2. Revert the commit
git revert <bad-commit-sha> --no-edit

# 3. Push the revert
git push origin main

# 4. ArgoCD detects the revert within 3 minutes and auto-syncs
argocd app wait user-service --sync --health --timeout 300
```

### Scenario 3: Emergency — Disable Auto-Sync

Use when: you need to stop ArgoCD from applying changes while you debug.

```bash
# Disable auto-sync for ALL apps (emergency)
for app in frontend user-service travel-service ai-service utility-service; do
  argocd app set $app --sync-policy none
done

# Re-enable after fix
for app in frontend user-service travel-service ai-service utility-service; do
  argocd app set $app --sync-policy automated
done
```

---

## Troubleshooting Sync Issues

### Issue: App is OutOfSync but changes look correct

```bash
# Check what ArgoCD thinks is different
argocd app diff user-service

# Force a hard refresh (clears ArgoCD's cached state)
argocd app get user-service --hard-refresh

# If using Helm: check if Helm release has manual changes
helm history user-service -n ai-travel
```

### Issue: Helm Rendering Error

```bash
# Test Helm rendering locally before pushing
helm template user-service infrastructure/helm-charts/user-service/ \
  --values infrastructure/helm-charts/user-service/values.yaml

# Common cause: missing required values, template syntax errors
# Fix the template and push to git
```

### Issue: "Resource already exists" Error

This happens when a Kubernetes resource was created manually (not via Helm) and ArgoCD tries to adopt it.

```bash
# Option A: Add the resource to Helm and annotate it
kubectl annotate <resource-type> <name> -n ai-travel \
  meta.helm.sh/release-name=user-service \
  meta.helm.sh/release-namespace=ai-travel
kubectl label <resource-type> <name> -n ai-travel \
  app.kubernetes.io/managed-by=Helm

# Option B: Delete the orphan resource and let ArgoCD recreate it
kubectl delete <resource-type> <name> -n ai-travel
argocd app sync user-service
```

### Issue: ArgoCD RBAC Permission Error

```bash
# Check ArgoCD logs for RBAC error details
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=50

# Check the ArgoCD project permissions
kubectl get appproject ai-travel -n argocd -o yaml

# Add missing namespace/resource permissions to the project
kubectl edit appproject ai-travel -n argocd
```

### Issue: Image Pull Fails in ArgoCD Sync

```bash
# Verify the ECR image exists
aws ecr describe-images \
  --repository-name ai-travel/user-service \
  --image-ids imageTag=sha-abc1234 \
  --region ap-south-1

# Check IRSA for image pull (pods use IRSA to pull from ECR)
kubectl describe pod <pod-name> -n ai-travel | grep -A 10 "Events:"

# Refresh ECR auth token (if using static imagePullSecrets)
# (IRSA-based pull does not require this — token auto-refreshes)
```
