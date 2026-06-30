# Plan My Journey — GitOps Operations Guide

## Overview

This project uses a **GitOps** methodology: the Git repository is the single source of truth for the desired state of all Kubernetes resources. ArgoCD continuously reconciles the live cluster state against what is defined in Git.

Key principles enforced:

- **No direct `kubectl apply`** for application changes — all changes flow through Git
- **Self-healing**: ArgoCD reverts manual `kubectl` changes within 3 minutes
- **Audit trail**: every change is a Git commit with author, timestamp, and diff
- **Rollback = git revert**: rolling back means reverting a commit, not running a script

---

## Repository Structure

The platform uses a **multi-repository** approach, with each concern in its own repository:

```
PlanMyJourney-App/                Application source code
│
├── services/
│   ├── user-service/             FastAPI microservice
│   ├── travel-service/           FastAPI microservice
│   ├── ai-service/               FastAPI + Bedrock integration
│   ├── ai-worker/                SQS consumer, KEDA-scaled
│   └── utility-service/          FastAPI microservice (weather, hotels, maps)
├── frontend/                     React + Vite
└── .github/workflows/            CI workflows (build, test, scan, push to ECR)

─────────────────────────────────────────────────────────────────────────

PlanMyJourney-Gitops/             GitOps manifests (source of truth for cluster state)
│
├── helm-charts/                  Helm chart definitions per service
│   ├── frontend/
│   │   ├── Chart.yaml
│   │   ├── templates/
│   │   ├── values-dev.yaml       Dev overrides (image tag updated by CI)
│   │   └── values-prod.yaml      Prod overrides (updated for promotions)
│   ├── user-service/
│   ├── travel-service/
│   ├── ai-service/
│   ├── ai-worker/
│   └── utility-service/
│
├── argocd-apps/                  ArgoCD Application manifests
│   ├── root-app.yaml             Root app-of-apps (entry point)
│   ├── projects/
│   │   └── planmyjourney.yaml    ArgoCD AppProject (permission boundaries)
│   └── applications/
│       ├── dev/                  Per-service Applications for dev namespace
│       └── prod/                 Per-service Applications for prod namespace
│
└── platform/                     Infrastructure platform components
    ├── argocd/                   ArgoCD Helm values
    ├── kgateway/                 KGateway Helm values + Gateway/HTTPRoute manifests
    ├── karpenter/                Karpenter Helm values + NodePool + EC2NodeClass
    ├── keda/                     KEDA Helm values + ScaledObjects + TriggerAuthentication
    └── monitoring/               kube-prometheus-stack Helm values

─────────────────────────────────────────────────────────────────────────

PlanMyjourney-Terraform/          AWS infrastructure (VPC, EKS, RDS, IAM, SQS, …)
│
└── .github/workflows/            Terraform plan/apply workflows

─────────────────────────────────────────────────────────────────────────

PlanMyJourney-Workflows/          Reusable GitHub Actions workflows
│
└── .github/workflows/
    ├── reusable-ci.yaml
    ├── reusable-push-ecr.yaml
    └── reusable-update-gitops.yaml

─────────────────────────────────────────────────────────────────────────

PlanMyJourney-Docs/               This documentation
```

---

## ArgoCD Concepts Used

### App-of-Apps Pattern

A single **root Application** (`root-app.yaml`) is the entry point. It points to `argocd-apps/applications/`. ArgoCD reads that directory and creates/manages all child Applications automatically.

```
root-app (app-of-apps, sync-wave -1)
    ├── platform/kgateway-app      (sync-wave 0)
    ├── platform/karpenter-app     (sync-wave 1)
    ├── platform/keda-app          (sync-wave 1)
    ├── platform/metrics-server    (sync-wave 1)
    ├── platform/monitoring-app    (sync-wave 2)
    ├── prod/frontend-app          (sync-wave 2)
    ├── prod/user-service-app      (sync-wave 2)
    ├── prod/travel-service-app    (sync-wave 2)
    ├── prod/ai-service-app        (sync-wave 2)
    ├── prod/ai-worker-app         (sync-wave 2)
    └── prod/utility-service-app   (sync-wave 2)
```

Adding a new service is as simple as adding a new `.yaml` file to `argocd-apps/applications/prod/` and pushing to Git.

### Sync Waves

Sync waves control the order of resource creation within a sync operation. Resources with a lower wave number are applied and must become healthy before higher wave resources are applied.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

### Auto-sync vs Manual Sync

| Mode | Behavior | When to Use |
|------|---------|------------|
| Auto-sync enabled | ArgoCD applies changes automatically on git push | Normal operation (all services) |
| Auto-sync disabled | Changes require `argocd app sync <name>` | During maintenance windows |
| Self-heal enabled | ArgoCD reverts manual `kubectl` changes | Always enabled in production |

### Pruning

When `prune: true` is set (default for all apps), ArgoCD deletes Kubernetes resources that exist in the cluster but are no longer present in Git. This prevents resource drift and zombie objects accumulating.

---

## Deployment Workflow (End-to-End)

```
Developer                 GitHub                    ArgoCD               Kubernetes
    │                        │                          │                     │
    │──push to main──────────►│                          │                     │
    │                        │──CI: lint, test, scan────►│                    │
    │                        │──CI: docker build─────────────────────────────►│
    │                        │──CI: push to ECR──────────────────────────────►│
    │                        │──CI: update values-dev.yaml in Gitops repo─────►│
    │                        │  (image.tag: <git-sha>)  │                     │
    │                        │                          │                     │
    │                        │◄─ArgoCD polls every 3min─│                     │
    │                        │                          │──detected drift──────►│
    │                        │                          │──helm upgrade────────►│
    │                        │                          │                     │──rolling update
    │                        │                          │◄─pods healthy────────│
    │                        │◄─sync complete───────────│                     │
```

### Step-by-Step

1. Developer pushes to `main` in `PlanMyJourney-App`
2. GitHub Actions CI pipeline runs: unit tests, SAST (SonarCloud), container scan (Trivy)
3. Docker image built with tag = full git SHA (e.g. `20cbe9b710d42565845c4673b8238a4ebc1b9a1e`)
4. Image pushed to ECR
5. CI commits updated `image.tag` to `PlanMyJourney-Gitops/helm-charts/<service>/values-dev.yaml`
6. ArgoCD polls the Gitops repo every 3 minutes (or receives webhook)
7. ArgoCD detects the diff in `values-dev.yaml` and runs `helm upgrade`
8. Kubernetes performs a rolling update — new pods start, old pods terminate after readiness probe passes
9. ArgoCD marks the app as `Synced + Healthy`

**Prod promotion** is a separate manual step — see [DEPLOYMENT.md — Prod Promotion](DEPLOYMENT.md#prod-promotion-manual-gitops).

---

## KEDA ScaledObjects in GitOps

KEDA ScaledObjects are managed by ArgoCD from `platform/keda/`. They define the event-driven scaling rules for the `ai-worker` and `ai-service` deployments.

### ai-worker — Scale to Zero on Empty Queue

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ai-worker-scaledobject
  namespace: prod
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  scaleTargetRef:
    name: ai-worker
  pollingInterval: 30
  cooldownPeriod: 120
  minReplicaCount: 0       # Scale to zero when no messages in queue
  maxReplicaCount: 10
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/235270183260/ai-travel-prod-ai-jobs
      queueLength: "5"     # 1 pod per 5 messages
      awsRegion: us-east-1
    authenticationRef:
      name: keda-trigger-auth-aws-irsa
```

### TriggerAuthentication — IRSA (no static credentials)

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-aws-irsa
  namespace: prod
spec:
  podIdentity:
    provider: aws-eks   # Uses the keda-operator service account's IRSA role
```

The IRSA role grants `sqs:GetQueueAttributes` to the KEDA operator. No AWS access keys are stored anywhere.

---

## ArgoCD CLI Cheatsheet

### Authentication

```bash
# Login using the public ArgoCD URL
argocd login argocd.invest-iq.online \
  --username admin \
  --password <password>

# Logout
argocd logout argocd.invest-iq.online
```

### Application Management

```bash
# List all applications and their status
argocd app list

# Get detailed status of a specific app
argocd app get ai-service-prod

# Sync manually
argocd app sync ai-service-prod

# Sync with pruning (delete removed resources)
argocd app sync ai-service-prod --prune

# Force sync (re-apply everything)
argocd app sync ai-service-prod --force

# Watch sync status in real-time
argocd app wait ai-service-prod --sync --health --timeout 300

# Diff: compare live cluster state vs git
argocd app diff ai-service-prod
```

### History and Rollback

```bash
# View deployment history
argocd app history ai-service-prod

# Rollback to a previous revision (by ID from history)
argocd app rollback ai-service-prod 2

# After rollback: re-enable auto-sync
argocd app set ai-service-prod --sync-policy automated
```

### Diagnostics

```bash
# Check why an app is out of sync
argocd app get ai-service-prod --show-operation

# Get the last sync operation result
argocd app get ai-service-prod -o json | jq '.status.operationState'

# List resources managed by an app
argocd app resources ai-service-prod

# Get pod logs via ArgoCD
argocd app logs ai-service-prod --container ai-service
```

---

## Adding a New Microservice

### Step 1: Create the Helm Chart

In `PlanMyJourney-Gitops`:

```bash
mkdir -p helm-charts/new-service/templates
```

Create `helm-charts/new-service/Chart.yaml`:

```yaml
apiVersion: v2
name: new-service
description: Description of new service
type: application
version: 0.1.0
appVersion: "1.0.0"
```

Create `helm-charts/new-service/values-prod.yaml`:

```yaml
replicaCount: 2

image:
  repository: <account-id>.dkr.ecr.us-east-1.amazonaws.com/planmyjourney/new-service
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8000

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 1

serviceAccount:
  name: new-service-sa
  irsaRoleArn: ""   # Set to Terraform output after creating IRSA role
```

### Step 2: Create the ArgoCD Application Manifest

In `argocd-apps/applications/prod/`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: new-service-prod
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: planmyjourney
  source:
    repoURL: https://github.com/Plan-My-Journey/PlanMyJourney-Gitops.git
    targetRevision: HEAD
    path: helm-charts/new-service
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Step 3: Add IRSA Role in Terraform

```hcl
# PlanMyjourney-Terraform/modules/eks/irsa.tf
module "new_service_irsa" {
  source    = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  role_name = "ai-travel-prod-new-service"

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["prod:new-service-sa"]
    }
  }
  # Attach policies as needed
}
```

### Step 4: Add GitHub Actions CI Workflow

In `PlanMyJourney-App/.github/workflows/`:

```bash
cp ci-user-service.yaml ci-new-service.yaml
# Edit: service name, Dockerfile path, ECR repository name
```

### Step 5: Push and Verify

```bash
git add helm-charts/new-service/ argocd-apps/applications/prod/new-service-prod.yaml
git commit -m "feat: add new-service to GitOps pipeline"
git push origin main

# Watch ArgoCD detect and sync the new app
argocd app list
argocd app wait new-service-prod --sync --health --timeout 300
```

---

## Rollback Procedure

### Scenario 1: Rollback via ArgoCD (Recommended)

```bash
# 1. Identify the last known good revision
argocd app history user-service-prod

# 2. Roll back
argocd app rollback user-service-prod <revision-id>

# 3. Monitor until healthy
argocd app wait user-service-prod --health --timeout 120

# 4. Verify pods are running
kubectl get pods -n prod -l app.kubernetes.io/name=user-service
kubectl logs -n prod -l app.kubernetes.io/name=user-service --tail=50
```

### Scenario 2: Git Revert

```bash
# In PlanMyJourney-Gitops:
git log --oneline helm-charts/<service>/values-prod.yaml
git revert <bad-commit-sha> --no-edit
git push origin main

# ArgoCD auto-syncs within 3 minutes
argocd app wait user-service-prod --sync --health --timeout 300
```

### Scenario 3: Emergency — Disable Auto-Sync

```bash
# Disable auto-sync to stop ArgoCD from applying changes while you debug
for app in frontend user-service travel-service ai-service ai-worker utility-service; do
  argocd app set ${app}-prod --sync-policy none
done

# Re-enable after the fix
for app in frontend user-service travel-service ai-service ai-worker utility-service; do
  argocd app set ${app}-prod --sync-policy automated
done
```

---

## Troubleshooting Sync Issues

### App is OutOfSync but changes look correct

```bash
# Check what ArgoCD thinks is different
argocd app diff user-service-prod

# Force a hard refresh (clears ArgoCD's cached state)
argocd app get user-service-prod --hard-refresh

# If using ServerSideApply: check for field manager conflicts
kubectl get deployment user-service -n prod -o yaml | grep managedFields
```

### Helm Rendering Error

```bash
# Test Helm rendering locally before pushing
helm template user-service helm-charts/user-service/ \
  --values helm-charts/user-service/values-prod.yaml

# Common cause: missing required values, template syntax errors
```

### "Resource already exists" Error

```bash
# Annotate and label the orphan resource so Helm/ArgoCD can adopt it
kubectl annotate deployment user-service -n prod \
  meta.helm.sh/release-name=user-service \
  meta.helm.sh/release-namespace=prod
kubectl label deployment user-service -n prod \
  app.kubernetes.io/managed-by=Helm

argocd app sync user-service-prod
```

### ArgoCD RBAC Permission Error

```bash
# Check ArgoCD controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=50

# Check the AppProject permissions
kubectl get appproject planmyjourney -n argocd -o yaml
```
