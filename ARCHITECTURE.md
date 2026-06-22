# AI-Travel-Planner: Architecture Overview

## System Architecture

### High-Level Overview

```
                            Internet
                               │
                               ▼
                        [Route53 DNS]
                    aitravel.com → ALB
                               │
                               ▼
                   ┌──────────────────────┐
                   │  AWS ALB (internet-  │
                   │  facing, HTTPS only) │
                   └──────────┬───────────┘
                              │  HTTPS :443
                              ▼
              ┌───────────────────────────────┐
              │         EKS Cluster           │
              │      (ap-south-1, v1.29)      │
              │                               │
              │  ┌─────────────────────────┐  │
              │  │   nginx-ingress         │  │
              │  │   (IngressClass: nginx) │  │
              │  └────────────┬────────────┘  │
              │               │               │
              │   ┌───────────┼───────────┐   │
              │   │           │           │   │
              │   ▼           ▼           ▼   │
              │ [frontend] [api-gw]  [argocd] │
              │              │               │
              │   ┌──────────┼───────────┐   │
              │   │          │           │   │
              │   ▼          ▼           ▼   │
              │ [user-svc] [travel-svc] [ai-svc]
              │   │          │           │   │
              │   │       [utility-svc]  │   │
              │   │                      │   │
              └───┼──────────────────────┼───┘
                  │                      │
                  ▼                      ▼
          [RDS PostgreSQL]      [AWS Bedrock]
           (Multi-AZ)           Nova Pro v1
        user_db + travel_db   (via VPC endpoint)
                               ap-south-1
```

### Request Flow

1. User browser → Route53 resolves `aitravel.com` → ALB DNS
2. ALB terminates HTTPS/TLS (ACM certificate), HTTP → HTTPS redirect
3. ALB → nginx-ingress controller pod (target group binding)
4. nginx-ingress routes by path prefix:
   - `/` → `frontend` service (Next.js SSR)
   - `/api/users/*` → `user-service` (FastAPI)
   - `/api/travel/*` → `travel-service` (FastAPI)
   - `/api/ai/*` → `ai-service` (FastAPI + Bedrock)
   - `/api/utility/*` → `utility-service` (FastAPI)
5. Backend services read secrets from Secrets Manager at startup (via IRSA)
6. `ai-service` calls Bedrock Converse API via VPC endpoint (never exits VPC)

---

## Networking Architecture

### VPC Design

```
VPC: 10.0.0.0/16  (65,536 addresses)
│
├── Public Subnets (ALB only — no pods)
│   ├── ap-south-1a: 10.0.1.0/24  (256 addresses)
│   └── ap-south-1b: 10.0.2.0/24  (256 addresses)
│
├── Private Subnets (EKS nodes + pods)
│   ├── ap-south-1a: 10.0.10.0/24  (256 addresses)
│   └── ap-south-1b: 10.0.11.0/24  (256 addresses)
│
└── Database Subnets (RDS — no internet route)
    ├── ap-south-1a: 10.0.20.0/24  (256 addresses)
    └── ap-south-1b: 10.0.21.0/24  (256 addresses)
```

### Traffic Routing

| Traffic Type | Path |
|-------------|------|
| Inbound HTTPS | Internet → ALB (public subnet) → nginx-ingress pod (private subnet) |
| Outbound (pods → internet) | Pod → NAT GW (public subnet) → Internet Gateway |
| AWS API calls (ECR, Secrets, Bedrock) | Pod → VPC Endpoint (stays in VPC, no NAT cost) |
| RDS connections | Pod (private subnet) → RDS (database subnet) — direct, no NAT |

### NAT Gateways

Two NAT Gateways are deployed (one per AZ) for high availability:

- `nat-gw-ap-south-1a` → attached to `10.0.1.0/24` (public subnet AZ-a)
- `nat-gw-ap-south-1b` → attached to `10.0.2.0/24` (public subnet AZ-b)

Private subnet route tables use the NAT GW in the same AZ for egress.

> **Cost note:** NAT Gateway data processing charges ($0.045/GB) are the single largest
> variable cost driver. VPC Endpoints eliminate NAT charges for AWS API traffic.

### VPC Endpoints

| Endpoint | Type | Services Covered |
|----------|------|-----------------|
| S3 Gateway | Gateway (free) | ECR image layers, S3 access |
| DynamoDB Gateway | Gateway (free) | Terraform state lock, FinOps DynamoDB |
| ECR API | Interface | `ecr.ap-south-1.amazonaws.com` |
| ECR DKR | Interface | `*.dkr.ecr.ap-south-1.amazonaws.com` |
| Secrets Manager | Interface | Secret reads at pod startup |
| SSM | Interface | Parameter Store access |
| SSM Messages | Interface | Session Manager (bastion-less access) |
| Bedrock Runtime | Interface | `bedrock-runtime.ap-south-1.amazonaws.com` |

### Security Groups

| SG Name | Inbound | Outbound |
|---------|---------|----------|
| `alb-sg` | 0.0.0.0/0:443, 0.0.0.0/0:80 | EKS nodes:30000-32767 |
| `eks-node-sg` | ALB SG:all, self | 0.0.0.0/0:all |
| `rds-sg` | EKS node SG:5432 | None |
| `vpc-endpoint-sg` | VPC CIDR:443 | None |

---

## Compute (EKS)

### Cluster Configuration

| Parameter | Value |
|-----------|-------|
| Kubernetes version | 1.29 |
| Control plane | AWS managed (private endpoint only) |
| OIDC provider | Enabled (required for IRSA) |
| Cluster logging | API, audit, authenticator → CloudWatch |
| CNI | AWS VPC CNI with prefix delegation |

### Node Group

| Parameter | Value |
|-----------|-------|
| Instance type | t3.medium (2 vCPU, 4 GB RAM) |
| AMI | Amazon Linux 2 (AL2) |
| Min nodes | 2 |
| Max nodes | 5 |
| Desired nodes | 2 |
| Disk | 50 GB gp3 EBS |
| Placement | Private subnets only |

### Kubernetes Add-ons

| Add-on | Version | Purpose |
|--------|---------|---------|
| aws-vpc-cni | latest | Pod networking (prefix delegation for higher pod density) |
| coredns | latest | Cluster DNS |
| kube-proxy | latest | iptables rules for services |
| aws-ebs-csi-driver | latest | Persistent volumes for stateful workloads |
| cluster-autoscaler | v1.29.x | Automatic node scaling |
| aws-load-balancer-controller | v2.7.x | ALB ingress management |

### IRSA (IAM Roles for Service Accounts)

Each service has a dedicated IAM role bound to its Kubernetes service account. Pods receive
temporary AWS credentials via the OIDC token projected into the pod (no static keys on nodes).

| Service Account | IAM Role Permissions |
|-----------------|---------------------|
| `user-service-sa` | `secretsmanager:GetSecretValue` (user-service secrets) |
| `travel-service-sa` | `secretsmanager:GetSecretValue` (travel-service secrets) |
| `ai-service-sa` | `bedrock:InvokeModel`, `secretsmanager:GetSecretValue` |
| `utility-service-sa` | `secretsmanager:GetSecretValue` (utility API keys) |
| `frontend-sa` | `secretsmanager:GetSecretValue` (frontend config) |
| `cluster-autoscaler-sa` | `autoscaling:*`, `ec2:DescribeInstanceTypes` |
| `alb-controller-sa` | `elasticloadbalancing:*`, `ec2:*`, `iam:CreateServiceLinkedRole` |
| `finops-sa` | `ce:*`, `compute-optimizer:*`, `cloudwatch:*`, `resourcegroupstagging:*` |

---

## AI Service Architecture

### Bedrock Integration

The `ai-service` is the core AI component responsible for generating travel plans, recommendations,
and conversational assistance.

```
[ai-service pod]
    │
    │  (OIDC token from service account)
    ▼
[AWS STS] ── assume role ──► [ai-service-irsa-role]
                                    │
                                    │  bedrock:InvokeModel
                                    ▼
                          [Bedrock Runtime API]
                         (via VPC Interface Endpoint)
                                    │
                                    ▼
                        [amazon.nova-pro-v1:0]
                           (ap-south-1 region)
```

### Converse API Usage

The service uses the Bedrock **Converse API** (not InvokeModel directly) for:
- Unified message format across models
- Built-in conversation history management
- Streaming response support
- Tool use / function calling

### Fallback Behavior

If Bedrock is unavailable (model access not granted, service outage), `ai-service` returns
pre-defined static travel recommendations rather than failing with a 500 error. This ensures
the frontend remains functional in degraded mode.

### Model Pricing (Nova Pro in ap-south-1)

| Token Type | Price |
|-----------|-------|
| Input | ~$0.0008 per 1,000 tokens |
| Output | ~$0.0032 per 1,000 tokens |
| Context window | 300,000 tokens |

Typical travel plan request: ~800 input tokens + ~500 output tokens = ~$0.002 per request.

---

## Database Architecture

### RDS PostgreSQL Configuration

| Parameter | Value |
|-----------|-------|
| Engine | PostgreSQL 15.4 |
| Instance class | db.t3.micro |
| Multi-AZ | Yes (automatic failover to standby) |
| Storage | 50 GB gp3, auto-scale to 100 GB |
| Encryption | AWS KMS (CMK) |
| Backup retention | 7 days |
| Backup window | 02:00–03:00 UTC |
| Maintenance window | Sun 04:00–05:00 UTC |
| Performance Insights | Enabled, 7-day retention |
| Enhanced Monitoring | 60-second intervals |

### Logical Databases

A single RDS instance hosts two logical databases, separated at the application level:

| Database | Owner Service | Schema Migration |
|----------|--------------|-----------------|
| `user_db` | user-service | Alembic (runs at pod startup) |
| `travel_db` | travel-service | Alembic (runs at pod startup) |

### Connection Management

```python
# SQLAlchemy engine configuration (per service)
engine = create_engine(
    DATABASE_URL,
    pool_size=10,           # Persistent connections
    max_overflow=20,        # Temporary burst connections
    pool_timeout=30,        # Wait time before "too many connections" error
    pool_pre_ping=True,     # Verify connection health before use
    pool_recycle=3600,      # Recycle connections every hour
)
```

Max simultaneous DB connections formula:
`max_connections = pods × (pool_size + max_overflow) = 2 × 30 = 60` (well within RDS limit of 87 for db.t3.micro)

---

## GitOps Architecture

### App-of-Apps Pattern

```
[Git Repository]
      │  (push / PR merge)
      │
      ▼
[GitHub Actions CI]
  ├── Run tests / lint
  ├── Build Docker image
  ├── Push to ECR
  └── Update image tag in helm-charts/*/values.yaml
                │
                │ (git commit [skip ci])
                ▼
[ArgoCD detects drift]
  │
  ├── Root Application (app-of-apps.yaml)
  │     └── Manages child Applications
  │           ├── frontend-app.yaml
  │           ├── user-service-app.yaml
  │           ├── travel-service-app.yaml
  │           ├── ai-service-app.yaml
  │           └── utility-service-app.yaml
  │
  └── Each child app: sync Helm chart → rolling update
```

### Sync Waves (Deployment Order)

| Wave | Resources | Purpose |
|------|-----------|---------|
| -1 | Namespaces, CRDs | Must exist before everything else |
| 0 | Secrets CSI driver, cert-manager | Infrastructure dependencies |
| 1 | nginx-ingress | Must be running before app ingresses |
| 2 | user-service, travel-service, utility-service | Core backend services |
| 3 | ai-service | Depends on other services for context |
| 4 | frontend | Depends on all backend APIs |

---

## Security Architecture

### Defense in Depth

```
Layer 1 — Network:   VPC subnets, Security Groups, NACLs
Layer 2 — TLS:       ALB terminates HTTPS, internal traffic HTTP (in-cluster)
Layer 3 — AuthN/Z:   IRSA (pod-level AWS identity), Kubernetes RBAC
Layer 4 — Secrets:   Secrets Manager (no env vars in YAML), CSI driver
Layer 5 — Runtime:   Pod security contexts, read-only root filesystem
Layer 6 — Supply Chain: ECR image scanning (Trivy), container signing
```

### Network Policies

Zero-trust NetworkPolicies are applied per namespace:

```yaml
# Only allow travel-service → RDS on port 5432
# Only allow ai-service → Bedrock endpoint on port 443
# Deny all ingress to pods except from nginx-ingress namespace
# Allow all pods egress to kube-dns for DNS resolution
```

### Secrets Management

All secrets are stored in AWS Secrets Manager and mounted into pods via the
**AWS Secrets and Configuration Provider (ASCP)** CSI driver. No secrets appear in:
- Kubernetes manifests
- Helm values files
- Environment variables in Deployments
- ECR image layers

### Encryption at Rest

| Resource | Encryption |
|----------|-----------|
| RDS | KMS CMK (`ai-travel-rds-key`) |
| EBS volumes (node disks) | KMS CMK (`ai-travel-ebs-key`) |
| Secrets Manager | KMS CMK (`ai-travel-secrets-key`) |
| CloudWatch Logs | KMS CMK (`ai-travel-logs-key`) |
| ECR images | AES-256 (AWS managed) |
| S3 (state, logs) | SSE-S3 / SSE-KMS |

---

## Cost Architecture

### FinOps Infrastructure

A dedicated Lambda function (`ai-travel-finops-cost-analyzer`) runs daily to analyze costs
and generate AI-powered optimization recommendations. See [FINOPS.md](FINOPS.md) for full details.

### Estimated Monthly Costs (ap-south-1, June 2026)

| Component | Configuration | Est. Monthly Cost |
|-----------|--------------|-------------------|
| EKS Cluster control plane | Managed | $73 |
| EKS Nodes | 2 × t3.medium | ~$28 |
| RDS PostgreSQL | db.t3.micro Multi-AZ | ~$55 |
| NAT Gateways | 2 × ap-south-1 (data ~50 GB/mo) | ~$65 |
| ALB | 1 internet-facing | ~$18 |
| ECR | 5 repos (~5 GB storage) | ~$5 |
| Secrets Manager | 6 secrets | ~$3 |
| CloudWatch | Logs + metrics + alarms | ~$10 |
| VPC Endpoints | 8 interface endpoints | ~$50 |
| KMS | 5 CMKs + API calls | ~$8 |
| S3 | State + logs | ~$3 |
| Lambda (FinOps) | Daily runs | <$1 |
| **Total (fixed)** | | **~$319/month** |
| **Bedrock (Nova Pro)** | Usage-based (est. 100K requests) | ~$0–$50/month |

> Costs are estimates for low-traffic production workloads. Heavy traffic increases NAT Gateway
> data transfer costs and Bedrock token costs. Reserved Instances for EKS nodes and RDS can reduce
> costs by 30–40%.

---

## Observability Architecture

### Three Pillars

**Logs** → CloudWatch Logs
- `/aws/eks/ai-travel-prod/cluster` — EKS control plane
- `/aws/eks/ai-travel-prod/application` — Application pod logs (via Fluent Bit DaemonSet)
- `/aws/lambda/ai-travel-finops-prod` — FinOps Lambda
- `ai-travel-alb-access-logs` — ALB access logs (S3)

**Metrics** → CloudWatch Metrics
- EKS: CPU, memory, pod counts via Container Insights
- RDS: CPU, connections, storage, IOPS via Enhanced Monitoring
- ALB: RequestCount, TargetResponseTime, HTTPCode_ELB_5XX
- Custom: Application-level metrics (request latency, error rate)

**Traces** → AWS X-Ray (optional, disabled by default)
- Distributed tracing across microservices
- Enable via `XRAY_ENABLED=true` environment variable

### CloudWatch Alarms

| Alarm | Threshold | Action |
|-------|-----------|--------|
| EKS node CPU | > 80% for 5 min | SNS → PagerDuty |
| RDS CPU | > 85% for 5 min | SNS → PagerDuty |
| RDS storage | < 20% free | SNS → PagerDuty |
| ALB 5xx errors | > 10/min | SNS → Slack |
| Pod restart rate | > 3 restarts/15 min | SNS → Slack |
| FinOps cost anomaly | > 15% vs 7-day avg | SNS → email |
