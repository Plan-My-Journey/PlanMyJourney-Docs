# AI-Travel-Planner: Architecture Overview

> Source of truth: this document is reconciled against the live Terraform
> (`PlanMyjourney-Terraform`, `environments/prod.tfvars`) and GitOps
> (`PlanMyJourney-Gitops`) repositories. Where a value is environment-specific it
> is taken from `prod.tfvars`.

## System Architecture

### High-Level Overview

```
                              Internet
                                 │
            invest-iq.online / www │ api.invest-iq.online
                                 ▼
                          [Route53 DNS]
                                 │
                                 ▼
                   ┌──────────────────────────┐
                   │  AWS NLB (internet-facing)│
                   │  TLS :443 terminated here │
                   │  (ACM cert, us-east-1)    │
                   └────────────┬─────────────┘
                                │  plain HTTP
                                ▼
              ┌───────────────────────────────────────┐
              │   EKS Cluster  "ai-travel-prod"        │
              │      (us-east-1, Kubernetes 1.30)      │
              │                                        │
              │  ns kgateway-system:                   │
              │   ┌──────────────────────────────┐     │
              │   │ Gateway "api-gateway"        │     │
              │   │ GatewayClass: kgateway       │     │
              │   │ (Envoy proxy; provisions NLB)│     │
              │   └──────────────┬───────────────┘     │
              │                  │  HTTPRoutes          │
              │                  │  (strip /api prefix) │
              │  ns prod:        ▼                      │
              │   /            → [frontend  :80]        │
              │   /api/auth,users → [user-service :8000]│
              │   /api/trips,expenses → [travel-svc:8000]
              │   /api/ai      → [ai-service :8000]     │
              │   /api/weather,hotels… → [utility:8000] │
              │                  │            │         │
              │   [ai-service] ──┘  SQS send  │         │
              │   [ai-worker] ◄ KEDA scales on queue    │
              │                                        │
              │  Platform (GitOps-managed):            │
              │   ArgoCD · KGateway · Karpenter · KEDA  │
              │   · metrics-server · kube-prometheus    │
              └───────┬───────────────────┬────────────┘
                      │                   │
                      ▼                   ▼
              [RDS PostgreSQL]      [AWS Bedrock — Nova Pro]
               db.t3.micro          (InvokeModel via IRSA;
               user_db + travel_db   egress via NAT, no VPC endpoint)
               (private DB subnets)
```

### Request Flow

1. User browser → Route53 resolves `invest-iq.online` (frontend) or
   `api.invest-iq.online` (API) → NLB DNS name.
2. **NLB terminates HTTPS/TLS** using the ACM certificate (us-east-1), then
   forwards **plain HTTP** to the KGateway Envoy proxy. The `Gateway` listener is
   declared `protocol: HTTP` on `port: 443` precisely because TLS is handled at
   the NLB (`gateway/base/gateway.yaml`).
3. KGateway `Gateway` "api-gateway" (namespace `kgateway-system`) routes to
   `HTTPRoute` objects in namespace `prod`.
4. HTTPRoutes match by path prefix and apply a `URLRewrite` filter that **strips
   `/api`** before forwarding (e.g. `/api/users` → `/users`). Longest-prefix wins,
   so `/api/*` beats the frontend catch-all `/`.
5. Request reaches the backend `Service` (`:8000` APIs, `:80` frontend) → Pod.
6. Backend services read secrets from Secrets Manager at startup (via IRSA).
7. `ai-service` calls the Bedrock Converse/InvokeModel API (Nova Pro) using its
   IRSA role; long-running generations are pushed to SQS and processed
   asynchronously by `ai-worker` (scaled by KEDA on queue depth).

> **Ingress note:** There is **no nginx-ingress and no ALB** in the live path.
> The legacy Terraform ALB module is disabled (`enable_legacy_alb = false`).
> Public ingress is Route53 → NLB → KGateway (Envoy) → Service → Pod. The NLB is
> provisioned automatically by KGateway from the `Gateway` + `GatewayParameters`
> objects, not by Terraform.

---

## Networking Architecture

### VPC Design (`modules/vpc`)

```
VPC: 10.0.0.0/16  (65,536 addresses)  — us-east-1
│
├── Public Subnets (NLB only — no pods)   tag kubernetes.io/role/elb=1
│   ├── us-east-1a: 10.0.1.0/24
│   └── us-east-1b: 10.0.2.0/24
│
├── Private Subnets (EKS nodes + pods)    tag kubernetes.io/role/internal-elb=1
│   ├── us-east-1a: 10.0.10.0/24
│   └── us-east-1b: 10.0.11.0/24
│
└── Database Subnets (RDS — no internet route)
    ├── us-east-1a: 10.0.20.0/24
    └── us-east-1b: 10.0.21.0/24
```

### Traffic Routing

| Traffic Type | Path |
|-------------|------|
| Inbound HTTPS | Internet → NLB (public subnet) → KGateway Envoy pod (private subnet) |
| Outbound (pods → internet) | Pod → NAT GW (public subnet) → Internet Gateway |
| AWS API calls (ECR, Secrets, SSM, S3, DynamoDB) | Pod → VPC Endpoint (stays in VPC, no NAT cost) |
| Bedrock calls | Pod → NAT GW → Internet (no Bedrock VPC endpoint configured) |
| RDS connections | Pod (private subnet) → RDS (database subnet) — direct, no NAT |

### NAT Gateways

Two NAT Gateways are deployed (one per AZ) for high availability, each with a
dedicated Elastic IP. Private subnet route tables send `0.0.0.0/0` to the NAT GW
in the same AZ. Database route tables have **no** `0.0.0.0/0` route (no internet).

> **Cost note:** NAT Gateway data processing is the largest variable cost driver.
> VPC Endpoints eliminate NAT charges for the AWS APIs they cover.

### VPC Endpoints (`modules/vpc`)

| Endpoint | Type | Purpose |
|----------|------|---------|
| S3 | Gateway (free) | ECR image layers, S3 access (attached to private + database RTs) |
| DynamoDB | Gateway (free) | FinOps tables / app DynamoDB access |
| Secrets Manager | Interface | Secret reads at pod startup |
| ECR API | Interface | `ecr.api` — auth/metadata for image pulls |
| ECR DKR | Interface | `ecr.dkr` — image layer pulls |
| SSM | Interface | Parameter Store / SSM access |
| SSM Messages | Interface | Session Manager control channel |
| EC2 Messages | Interface | Session Manager / SSM agent channel |

> There is intentionally **no Bedrock interface endpoint** today; `ai-service`
> reaches Bedrock through the NAT Gateway. Adding a `bedrock-runtime` interface
> endpoint is a possible future optimization to keep that traffic in-VPC.

### Security Groups (`modules/vpc` + service modules)

| SG | Inbound | Outbound |
|----|---------|----------|
| `ai-travel-prod-alb-sg` | 0.0.0.0/0:80, 0.0.0.0/0:443 | VPC CIDR (all) |
| `ai-travel-prod-eks-cluster-sg` | 443 from alb-sg; all from self (node-to-node) | 0.0.0.0/0 (all) |
| `ai-travel-prod-vpc-endpoints-sg` | 443 from VPC CIDR | 0.0.0.0/0 |
| RDS SG (`modules/rds`) | 5432 from EKS node/cluster SG | — |

---

## Compute (EKS)

### Cluster Configuration (`modules/eks`, `prod.tfvars`)

| Parameter | Value |
|-----------|-------|
| Cluster name | `ai-travel-prod` |
| Kubernetes version | 1.30 |
| OIDC provider | Enabled (required for IRSA and Karpenter) |
| Worker placement | Private subnets only |

### Baseline Managed Node Group

| Parameter | Value |
|-----------|-------|
| Instance type | t3.small |
| Min / Desired / Max | 2 / 2 / 5 |
| Disk | 50 GB EBS |

The managed node group is the **baseline** that runs cluster-critical add-ons
(ArgoCD, KGateway, Karpenter itself, KEDA, metrics-server, monitoring). Dynamic
application capacity is provided by **Karpenter**.

### Karpenter NodePool (`PlanMyJourney-Gitops/platform/karpenter`)

| Parameter | Value |
|-----------|-------|
| EC2NodeClass | `default`, AMI family AL2 (`al2@latest`), node role `ai-travel-prod-karpenter-node` |
| Subnets | selected by tag `kubernetes.io/role/internal-elb=1` (private) |
| Capacity types | `spot` + `on-demand` |
| Instance types | t3.large, t3.xlarge, m5.large, m5.xlarge, m5a.large, m5a.xlarge |
| Disruption | `WhenEmptyOrUnderutilized`, consolidate after 1m |
| Limits | cpu 32, memory 64Gi; nodes expire after 720h |

> Larger instance types are used deliberately: CPU is barely used, but small
> types like t3.small cap at ~11 pods (ENI/prefix limits). Bigger nodes pack more
> pods, so Karpenter consolidates onto fewer nodes.

### IRSA (IAM Roles for Service Accounts) (`modules/irsa`, `modules/karpenter`)

Each workload that needs AWS access has a dedicated IAM role bound to its
Kubernetes service account via the cluster OIDC provider. Pods receive temporary
credentials through the projected OIDC token — no static keys on nodes.

| Service Account | Permissions (intent) |
|-----------------|----------------------|
| `ai-service` SA | `bedrock:InvokeModel`, Secrets Manager read, SQS send |
| `ai-worker` SA | SQS receive/delete, DynamoDB jobs table, Secrets Manager read |
| `user-service` / `travel-service` / `utility-service` / `frontend` SAs | Secrets Manager read (own secrets) |
| Karpenter controller SA | EC2 provisioning, pricing, instance profile, SQS interruption queue |

---

## AI Service Architecture

### Bedrock Integration

`ai-service` generates travel plans and recommendations. Synchronous calls hit
Bedrock directly; long-running work is offloaded to an async pipeline.

```
[ai-service pod] ──OIDC token──► [STS AssumeRoleWithWebIdentity]
       │                                   │
       │                                   ▼
       │                         [ai-service IRSA role]
       │                                   │ bedrock:InvokeModel
       │                                   ▼
       │                         [Bedrock Runtime — Nova Pro]
       │
       └── for long jobs ──► [SQS queue] ──► [ai-worker pods]
                                   ▲                │ KEDA scales worker
                                   └── queue depth ─┘ on SQS backlog (incl. zero)
```

### Async Jobs (SQS + KEDA)

`ai-worker` (`helm-charts/ai-worker`) consumes the SQS queue created in
`modules/sqs`. A KEDA `ScaledObject` scales the worker on queue depth, including
**scale-to-zero** when the queue is empty. Job state is tracked in a DynamoDB
jobs table. IRSA grants the worker SQS + DynamoDB + Secrets access.

### Fallback Behavior

If Bedrock is unavailable (model access not granted, outage), `ai-service`
returns pre-defined static recommendations rather than a 500, keeping the
frontend functional in degraded mode.

---

## Database Architecture

### RDS PostgreSQL Configuration (`modules/rds`, `prod.tfvars`)

| Parameter | Value |
|-----------|-------|
| Engine | PostgreSQL |
| Instance class | db.t3.micro |
| Multi-AZ | **Disabled by default** (`enable_rds_multi_az = false`); enable in a planned maintenance window |
| Storage | 50 GB, autoscale to 100 GB |
| Encryption | AWS KMS CMK (from `modules/secrets`) |
| Backup retention | 30 days |
| Credentials | master password stored in Secrets Manager (`modules/secrets`), never in code |
| Placement | Database subnets (no internet route) |

### Logical Databases

A single RDS instance hosts the application data; services own their tables and
run **Alembic** migrations at pod startup.

| Owner Service | Migration |
|--------------|-----------|
| user-service | Alembic (`services/user-service/alembic`) |
| travel-service | Alembic (`services/travel-service/alembic`) |

> Note: the DB master password must be alphanumeric — a percent-encoded password
> breaks Alembic startup (URL parsing). See TROUBLESHOOTING.md.

---

## GitOps Architecture

### App-of-Apps Pattern (`PlanMyJourney-Gitops/argocd-apps`)

```
[GitHub: PlanMyJourney-App]
      │  push / PR merge
      ▼
[GitHub Actions CI]  (reusable workflows in PlanMyJourney-Workflows)
  ├── lint, SAST, SCA (quality gates)
  ├── docker build → push to ECR
  └── commit new image tag into PlanMyJourney-Gitops (Helm values)
                │
                ▼
[ArgoCD]  root Application "planmyjourney-app-of-apps" (sync-wave -1)
  └── recurses argocd-apps/applications/ →
        ├── infrastructure/  (gateway, kgateway[-crds], karpenter, keda,
        │                      metrics-server, monitoring)
        ├── dev/   (per-service Applications)
        └── prod/  (per-service Applications)
  syncPolicy: automated { prune, selfHeal }, ServerSideApply, CreateNamespace
```

CI never runs `kubectl` — it ends at Git. ArgoCD (in-cluster) reconciles the
desired state, so no CI runner holds cluster credentials (pull-based GitOps).

### Sync Waves (ordering)

Sync waves order platform CRDs/controllers before workloads (e.g. the app-of-apps
root is wave `-1`; gateway/CRDs and controllers come before application services).

---

## Security Architecture

### Defense in Depth

```
Layer 1 — Network:   private subnets, Security Groups, DB isolation
Layer 2 — TLS:       NLB terminates HTTPS (ACM); in-cluster traffic is HTTP
Layer 3 — Identity:  IRSA (pod-level AWS identity via OIDC+STS); GitHub OIDC for CI
Layer 4 — AuthN:     AWS Cognito user pool (hosted UI, OAuth callback/logout URLs)
Layer 5 — Secrets:   Secrets Manager + KMS; no secrets in YAML/images/env
Layer 6 — Policy:    NetworkPolicies per service; PodDisruptionBudgets
Layer 7 — Supply chain: CI SAST/SCA gates + ECR image scanning
```

### GitHub OIDC for CI

GitHub Actions assumes an AWS IAM role via OIDC (no long-lived cloud keys). The
allowed repos are pinned in `github_repos` and are **case-sensitive** — the OIDC
`sub` claim and IAM `StringLike` matching are case-sensitive, so the canonical
`Plan-My-Journey/...` casing must be used or STS returns "Not authorized".

### Cognito (`modules/cognito`)

User authentication uses a Cognito user pool with hosted UI domain prefix
`planmyjourney`. Callback URLs: `https://invest-iq.online/callback` (+ localhost
for dev); logout URLs mirror these.

### Encryption at Rest

| Resource | Encryption |
|----------|-----------|
| RDS | KMS CMK (`modules/secrets`) |
| Secrets Manager | KMS CMK |
| EBS volumes / EKS | KMS CMK |
| ECR images | AES-256 (AWS managed) |
| S3 (Terraform state, logs) | SSE |

---

## Cost Architecture

### FinOps Infrastructure (`modules/finops`)

A Lambda function (`cost-anomaly-detector-prod`) runs on an EventBridge schedule
to analyze costs and email reports (SES verified sender), with baselines stored
in DynamoDB (`finops-cost-baselines`). See [FINOPS.md](FINOPS.md).

### Major Cost Drivers (us-east-1, low-traffic prod)

| Component | Configuration |
|-----------|---------------|
| EKS control plane | managed (~$73/mo) |
| Baseline nodes | 2 × t3.small + Karpenter (spot-preferred) dynamic capacity |
| RDS | db.t3.micro (single-AZ by default) |
| NAT Gateways | 2 (largest variable cost; mitigated by VPC endpoints) |
| VPC interface endpoints | 6 interface + 2 gateway |
| Bedrock | usage-based (Nova Pro tokens) |

> Karpenter prefers spot, consolidates aggressively, and uses larger instances
> for pod density. KEDA scale-to-zero removes idle `ai-worker` cost.

---

## Observability Architecture

**In-cluster metrics** → `kube-prometheus-stack` (Prometheus + Grafana), deployed
via the `monitoring` ArgoCD Application. Each service Helm chart ships a
`ServiceMonitor` so Prometheus scrapes it; Grafana is exposed via
`monitoring/grafana-route.yaml`. `metrics-server` provides resource metrics for
HPA.

**AWS-side** → CloudWatch logs/metrics with SNS alarm notifications to the
configured `alert_email` (`modules/monitoring`). Log retention is 7 days
(`log_retention_days`).

**Cost monitoring** → FinOps Lambda (daily/scheduled) with email reports
(`modules/finops`).
