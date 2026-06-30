# Plan My Journey — AI-Powered Travel Planner

A production-grade, cloud-native travel planning platform built on AWS. Plan My Journey lets users generate personalised AI itineraries, track trip budgets and expenses, compare destinations, and get smart packing recommendations — all served from a secure, fully automated Kubernetes infrastructure.

**Live URL:** https://invest-iq.online

---

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Services & APIs](#services--apis)
- [Frontend Application](#frontend-application)
- [Authentication](#authentication)
- [Infrastructure (AWS)](#infrastructure-aws)
- [Kubernetes & GitOps](#kubernetes--gitops)
- [CI/CD Pipelines](#cicd-pipelines)
- [Observability & Monitoring](#observability--monitoring)
- [Security](#security)
- [Local Development](#local-development)
- [Deployment](#deployment)
- [Scaling](#scaling)
- [Disaster Recovery](#disaster-recovery)
- [Cost Management (FinOps)](#cost-management-finops)
- [API Documentation](#api-documentation)
- [Detailed Documentation](#detailed-documentation)

---

## Overview

Plan My Journey is a microservices-based SaaS platform deployed on Amazon EKS. It uses AWS Bedrock (Amazon Nova Pro) to generate real-time AI travel itineraries, and exposes a React single-page application served through a Kubernetes-native API gateway.

**Core Features:**
- AI-generated day-by-day itineraries with morning, afternoon, and evening activities
- Budget optimisation suggestions for any destination
- Side-by-side destination comparison
- Intelligent packing list generation
- Expense tracker per trip
- Geocoding, weather, and hotel search via third-party APIs
- User authentication via AWS Cognito (OAuth2) or local JWT

---

## Repository Structure

The project is split across five GitHub repositories under the [`Plan-My-Journey`](https://github.com/Plan-My-Journey) organisation:

| Repository | Purpose |
|---|---|
| [PlanMyJourney-App](https://github.com/Plan-My-Journey/PlanMyJourney-App) | Application source code — React frontend + four FastAPI backend microservices |
| [PlanMyjourney-Terraform](https://github.com/Plan-My-Journey/PlanMyjourney-Terraform) | AWS infrastructure as code — 16 Terraform modules (EKS, RDS, VPC, Cognito, IAM, SQS, etc.) |
| [PlanMyJourney-Gitops](https://github.com/Plan-My-Journey/PlanMyJourney-Gitops) | Kubernetes manifests, Helm charts, and ArgoCD application definitions |
| [PlanMyJourney-Workflows](https://github.com/Plan-My-Journey/PlanMyJourney-Workflows) | 30+ reusable GitHub Actions workflows (SAST, SCA, Docker build, ArgoCD sync) |
| [PlanMyJourney-Docs](https://github.com/Plan-My-Journey/PlanMyJourney-Docs) | Architecture documentation and runbooks (this repo) |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                            INTERNET                                  │
└───────────────────────────────┬──────────────────────────────────────┘
                                │ HTTPS (443)
                     ┌──────────▼──────────┐
                     │   AWS Route 53      │
                     │   (DNS)             │
                     └──────────┬──────────┘
                                │
                     ┌──────────▼──────────┐
                     │   AWS NLB           │
                     │   (TLS Termination, │
                     │    ACM Certificate) │
                     └──────────┬──────────┘
                                │ HTTP (in-VPC)
                     ┌──────────▼──────────┐
                     │   KGateway          │
                     │   (Envoy Proxy,     │
                     │    HTTPRoutes)      │
                     └──┬─────────────┬───┘
              /          │             │ /api/*
    ┌─────────▼──────┐   │   ┌────────▼───────────────────────┐
    │   Frontend     │   │   │   Backend Services              │
    │   (React SPA,  │   │   │   ┌────────────────────────┐   │
    │    Nginx)      │   │   │   │ user-service (FastAPI)  │   │
    └────────────────┘   │   │   ├────────────────────────┤   │
                         │   │   │ travel-service (FastAPI)│   │
              Kubernetes │   │   ├────────────────────────┤   │
              (EKS)      │   │   │ ai-service (FastAPI +  │   │
                         │   │   │  AWS Bedrock)          │   │
                         │   │   ├────────────────────────┤   │
                         │   │   │ utility-service        │   │
                         │   │   │ (FastAPI + httpx)      │   │
                         │   │   └────────────────────────┘   │
                         │   └────────────────────────────────┘
                         │
          ┌──────────────┼───────────────────────────────┐
          │                   AWS Services                │
          │  ┌──────────┐  ┌───────────┐  ┌──────────┐  │
          │  │  RDS     │  │  Cognito  │  │  Bedrock │  │
          │  │  (PG)    │  │  (Auth)   │  │  (AI)    │  │
          │  └──────────┘  └───────────┘  └──────────┘  │
          │  ┌──────────┐  ┌───────────┐  ┌──────────┐  │
          │  │  SQS     │  │  DynamoDB │  │  Secrets │  │
          │  │  (Queue) │  │  (Jobs)   │  │  Manager │  │
          │  └──────────┘  └───────────┘  └──────────┘  │
          └──────────────────────────────────────────────┘
```

**Traffic flow:**
1. User visits `https://invest-iq.online`
2. Route 53 resolves to the NLB public IP
3. NLB terminates TLS using an ACM certificate and forwards plain HTTP into the VPC
4. KGateway (Envoy-backed Kubernetes Gateway API) routes the request:
   - `/` → `frontend` service (React SPA served by Nginx)
   - `/api/auth`, `/api/users` → `user-service`
   - `/api/trips`, `/api/expenses` → `travel-service`
   - `/api/ai` → `ai-service` (calls Bedrock or returns fallback)
   - `/api/weather`, `/api/hotels`, `/api/places`, `/api/geocode`, `/api/routing` → `utility-service`
5. Services authenticate users via JWT (signed by Cognito or local secret)
6. Services access AWS resources (Bedrock, Secrets Manager, SQS) via IRSA — no static credentials

---

## Tech Stack

### Frontend
| Technology | Version | Role |
|---|---|---|
| React | 18.3 | UI framework |
| TypeScript | 5.x | Type safety |
| Vite | 6.0 | Build tool & dev server |
| TailwindCSS | 3.4 | Utility-first styling |
| Axios | 1.x | HTTP client (45s timeout) |
| React Router | 6 | Client-side routing |
| Lucide React | — | Icon library |
| Nginx | 1.27-alpine | Production static file server |

### Backend (all services)
| Technology | Version | Role |
|---|---|---|
| Python | 3.12 | Runtime |
| FastAPI | 0.115 | API framework |
| Uvicorn + Gunicorn | 0.34 / 21.2 | ASGI server |
| SQLAlchemy | 2.0 | ORM (user-service, travel-service) |
| Alembic | 1.14 | Database migrations |
| Psycopg | 3.2 | PostgreSQL async driver |
| pydantic-settings | 2.7 | Environment-based config |
| python-jose | 3.4 | JWT creation and validation |
| passlib[bcrypt] | 1.7 | Password hashing |
| slowapi | — | Rate limiting |

### AI Service specific
| Technology | Role |
|---|---|
| AWS Bedrock (Amazon Nova Pro) | LLM for travel content generation |
| boto3 | AWS SDK (Bedrock Converse API) |
| AWS SQS | Optional async job queue |
| AWS DynamoDB | Async job state tracking |

### Utility Service specific
| Technology | Role |
|---|---|
| httpx | Async HTTP client |
| OpenWeather API | Weather data |
| Geoapify API | Geocoding, hotels, places, routing |

### Infrastructure
| Technology | Role |
|---|---|
| AWS EKS | Managed Kubernetes |
| AWS RDS (PostgreSQL) | Relational database |
| AWS Cognito | User authentication & OAuth2 |
| AWS Secrets Manager | Credentials management |
| AWS KMS | Encryption at rest |
| AWS ECR | Container image registry |
| AWS SQS | Async message queue |
| AWS DynamoDB | Key-value store (jobs, FinOps baselines) |
| AWS CloudTrail | Audit logging |
| AWS CloudWatch | Metrics and log aggregation |
| AWS VPC Endpoints | Private AWS API access (no NAT for SSM, ECR, S3) |
| Terraform | Infrastructure as code |
| Helm | Kubernetes package manager |
| ArgoCD | GitOps continuous delivery |
| Karpenter | Just-in-time node provisioning with spot/on-demand mix |
| KEDA | Event-driven pod autoscaling based on SQS queue depth |
| Prometheus + Grafana | In-cluster metrics and dashboards |
| GitHub Actions | CI/CD automation |

---

## Services & APIs

All services share:
- FastAPI app structure (`main.py`, `api/routes/`, `schemas/`, `core/config.py`)
- Health check endpoints: `GET /health`, `GET /healthz`, `GET /ready`
- JWT authentication required on all routes except `/auth/login` and `/auth/register`
- Rate limiting via `slowapi`
- CORS configured via `CORS_ORIGINS` environment variable

### User Service
Manages registration, login, and user profile.

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/login` | Email + password → JWT token |
| `POST` | `/auth/register` | Create new user account → JWT token |
| `GET` | `/users/{id}` | Get user profile |
| `PUT` | `/users/{id}` | Update user profile |
| `DELETE` | `/users/{id}` | Delete user account |

**Database:** PostgreSQL (`user_db`). Migrations managed by Alembic, applied automatically on pod startup.

### Travel Service
Manages trip creation, expense tracking, and budget summaries.

| Method | Path | Description |
|---|---|---|
| `POST` | `/trips` | Create a new trip |
| `GET` | `/trips` | List all trips for the authenticated user |
| `GET` | `/trips/{id}` | Get trip details |
| `PUT` | `/trips/{id}` | Update trip |
| `DELETE` | `/trips/{id}` | Delete trip |
| `GET` | `/trips/budget-summary` | Total budget, spent, and remaining across all trips |
| `GET` | `/trip-history` | Alias for trip listing (used by frontend history page) |
| `POST` | `/expenses` | Log an expense |
| `GET` | `/expenses/{trip_id}` | List expenses for a trip |
| `PUT` | `/expenses/{id}` | Update expense |
| `DELETE` | `/expenses/{id}` | Delete expense |

**Database:** PostgreSQL (`travel_db`). Tables: `trips`, `expenses`.

### AI Service
Generates AI travel content using AWS Bedrock (Amazon Nova Pro model). Supports both synchronous and asynchronous (SQS-backed) modes.

| Method | Path | Description |
|---|---|---|
| `POST` | `/ai/itinerary` | Generate a day-by-day trip itinerary |
| `POST` | `/ai/chat` | Ask a travel question, get AI answer |
| `POST` | `/ai/budget-optimizer` | Get budget saving suggestions |
| `POST` | `/ai/compare` | Compare two destinations |
| `POST` | `/ai/packing-list` | Generate destination-specific packing list |
| `GET` | `/ai/jobs/{job_id}` | Poll status of an async job |

**AI mode:**
- `ASYNC_JOBS_ENABLED=false` (production default): Bedrock is called directly, response returned synchronously (HTTP 200).
- `ASYNC_JOBS_ENABLED=true`: Job enqueued to SQS (HTTP 202), processed by `ai-worker` pod, polled via `/ai/jobs/{job_id}`.

**Bedrock integration:**
- Model: `amazon.nova-pro-v1:0`
- API: Bedrock Converse API (model-agnostic, handles retries and marshalling)
- Credentials: IRSA — no static AWS keys in pods
- Fallback: Static curated responses for common destinations ensure the endpoint never returns 5xx

**Sample request — itinerary:**
```json
POST /api/ai/itinerary
{
  "destination": "Varkala",
  "budget": 3000,
  "days": 3,
  "interests": ["beach", "culture"],
  "status": "planned"
}
```

**Sample response:**
```json
{
  "trip_summary": "3 days in Varkala...",
  "day_wise_plan": [
    {
      "day": 1,
      "title": "Papanasam Beach & Cliffs",
      "morning": "...",
      "afternoon": "...",
      "evening": "..."
    }
  ],
  "estimated_budget_breakdown": {
    "lodging": "$1200.00",
    "food": "$750.00",
    "activities": "$600.00",
    "transport": "$450.00"
  },
  "travel_tips": ["..."]
}
```

### Utility Service
Provides weather, geocoding, and points-of-interest data from third-party APIs.

| Method | Path | Description |
|---|---|---|
| `GET` | `/weather/{city}` | Current weather (OpenWeather API) |
| `GET` | `/hotels/{city}` | Hotel listings (Geoapify) |
| `GET` | `/places/{city}` | Points of interest (Geoapify) |
| `GET` | `/geocode/autocomplete` | Location autocomplete (`?text=...&limit=5`) |
| `GET` | `/geocode/search` | Location coordinate lookup |
| `GET` | `/routing` | Route between two coordinates (`?origin_lat&origin_lon&destination_lat&destination_lon&mode=drive`) |

---

## Frontend Application

A React 18 TypeScript SPA built with Vite and styled with TailwindCSS. Served in production by Nginx inside a Docker container on the EKS cluster.

### Pages

| Route | Component | Description |
|---|---|---|
| `/login` | `Login.tsx` | Email/password login form |
| `/register` | `Register.tsx` | New user registration |
| `/callback` | `CognitoCallback.tsx` | Handles Cognito OAuth redirect with `code` parameter |
| `/` | `Dashboard.tsx` | Overview — current trips, AI quick-start |
| `/create-trip` | `CreateTrip.tsx` | AI-powered trip creation form (destination, budget, days, interests) |
| `/trips` | `TripHistory.tsx` | All past and upcoming trips |
| `/expenses` | `ExpenseTracker.tsx` | Per-trip expense logging and summary |
| `/ai` | `AITravelAssistant.tsx` | Conversational AI travel assistant |
| `/compare` | `DestinationComparison.tsx` | Side-by-side destination comparison |
| `/packing` | `PackingAssistant.tsx` | Smart packing checklist generator |
| `/profile` | `UserProfile.tsx` | User account settings |

### State Management

React Context (no Redux or Zustand):
- `AuthContext` — stores authenticated user and JWT, handles login/logout
- `TripContext` — active trip and trips list
- `AppContext` — global loading state and notifications

### API Integration

```typescript
// src/api/client.ts
const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 45000,
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem("ai_travel_token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

Async job polling (`src/api/asyncJobs.ts`): when the AI service returns HTTP 202, the frontend polls `/ai/jobs/{job_id}` every 2 seconds (up to 60 attempts = 120 seconds) until the job completes or fails.

### Environment Variables (Vite)

```env
VITE_API_BASE_URL=https://api.invest-iq.online/api
VITE_COGNITO_REGION=us-east-1
VITE_COGNITO_USER_POOL_ID=<pool-id>
VITE_COGNITO_CLIENT_ID=<app-client-id>
VITE_COGNITO_DOMAIN=planmyjourney.auth.us-east-1.amazoncognito.com
VITE_COGNITO_REDIRECT_URI=https://invest-iq.online/callback
```

---

## Authentication

### User Authentication (Cognito OAuth2)

**Production flow:**
1. Frontend redirects to Cognito Hosted UI (`/oauth2/authorize`)
2. User authenticates with email/password
3. Cognito redirects back to `/callback?code=...`
4. Frontend exchanges the code for `id_token`, `access_token`, `refresh_token`
5. Tokens stored in `localStorage`; `access_token` added to every request via Axios interceptor

**Backend validation:**
- Services fetch the Cognito JWKS endpoint and verify the RS256 token signature
- Token audience (Cognito App Client ID) and issuer verified on every request
- Falls back to local HS256 JWT validation in non-production environments

### Service Identity (IRSA)

All pods that call AWS services (Bedrock, Secrets Manager, SQS, DynamoDB) use IAM Roles for Service Accounts. The pod's Kubernetes service account is annotated with an IAM role ARN; EKS exchanges the pod's OIDC token for temporary STS credentials. No static access keys exist anywhere in the codebase or cluster.

**Per-service IAM permissions:**
- `ai-service` — `bedrock:InvokeModel`, `sqs:SendMessage`, `secretsmanager:GetSecretValue`
- `ai-worker` — `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `dynamodb:PutItem`, `dynamodb:GetItem`
- `user-service`, `travel-service`, `utility-service`, `frontend` — `secretsmanager:GetSecretValue` (own secret paths only, scoped by resource condition)
- `karpenter` — EC2 `CreateFleet`, `RunInstances`, `TerminateInstances`, `DescribeInstances`, `iam:PassRole` (scoped to Karpenter node role)
- `keda-operator` — `sqs:GetQueueAttributes`, `cloudwatch:GetMetricStatistics`

### GitHub Actions (OIDC)

CI/CD pipelines assume AWS IAM roles using GitHub's OIDC provider — no long-lived AWS credentials stored in GitHub secrets. The trust policy restricts access to `Plan-My-Journey/*` org repositories on the `main` branch.

---

## Infrastructure (AWS)

All infrastructure is managed by Terraform in the [`PlanMyjourney-Terraform`](https://github.com/Plan-My-Journey/PlanMyjourney-Terraform) repository. State is stored in S3 with DynamoDB locking and KMS encryption.

### Terraform Code Quality

| Standard | Detail |
|---|---|
| Validation | `terraform validate` and `terraform fmt` pass with zero errors on every PR |
| No hardcoded values | All environment-specific values declared in `environments/prod.tfvars`; no literals in `.tf` files |
| Resource tagging | Every AWS resource carries `Environment = "production"` and `Owner = "planmyjourney"` tags via a common `locals.tf` tag map |
| Remote state | S3 bucket (`ai-travel-prod-terraform-state`) with versioning and KMS encryption; DynamoDB table (`ai-travel-prod-terraform-locks`) for state locking |
| `.gitignore` | Excludes `*.tfstate`, `*.tfstate.backup`, `.terraform/`, `.env`, `terraform.tfvars`, and `credentials` files — no secrets ever committed |
| Terraform outputs | `outputs.tf` documents cluster endpoint, OIDC provider ARN, ECR URLs, RDS endpoint, SQS queue URL, and Secrets Manager secret ARNs |

### Terraform Modules

| Module | AWS Resources Created |
|---|---|
| `vpc` | VPC, public / private / database subnets across 2 AZs, NAT Gateways, Internet Gateway, route tables, security groups |
| `eks` | EKS cluster, managed node group (t3.medium), OIDC provider, KMS encryption for secrets |
| `rds` | PostgreSQL Multi-AZ instance, subnet group, parameter group, KMS CMK, 30-day automated backups, deletion protection |
| `iam` | Cluster role, node role, GitHub OIDC federation role |
| `ecr` | ECR repositories for all 5 services, KMS encryption, lifecycle policies (keep last 20 images) |
| `secrets` | Secrets Manager entries (RDS password, API keys), KMS CMK |
| `cognito` | Cognito User Pool, App Client, Hosted UI domain, callback/logout URLs |
| `irsa` | IAM roles for each Kubernetes service account (trust policy via OIDC) |
| `karpenter` | Karpenter IRSA role (`ai-travel-prod-karpenter-node`), NodePool, EC2NodeClass, subnet/SG discovery tags |
| `keda` | KEDA operator IRSA role (`ai-travel-prod-keda-operator`), SQS queue access policy |
| `sqs` | SQS queue for async AI jobs, DLQ, DynamoDB table for job state, KMS encryption |
| `monitoring` | CloudWatch log groups (7-day retention), SNS topics, RDS and cluster alarms |
| `governance` | CloudTrail, AWS Config rules, cost allocation tags |
| `finops` | Lambda (cost anomaly detector), EventBridge hourly schedule, DynamoDB baselines table, SES email reports |
| `alb` | (Disabled in production — `enable_legacy_alb = false`) |
| `frontend-hosting` | (Unused; frontend served in-cluster via Nginx) |

### Network Layout

**Region:** us-east-1 | **VPC CIDR:** 10.0.0.0/16 | **Availability Zones:** us-east-1a, us-east-1b

```
VPC (10.0.0.0/16)
├── Public Subnets     (10.0.1.0/24, 10.0.2.0/24)    — NLB only
├── Private Subnets    (10.0.10.0/24, 10.0.11.0/24)  — EKS nodes + pods
└── Database Subnets   (10.0.20.0/24, 10.0.21.0/24)  — RDS only (no internet route)
```

**Traffic paths:**

| Direction | Path |
|---|---|
| Inbound HTTPS | Internet → NLB (public) → KGateway (private) → service pod |
| Pod → Bedrock / NAT | Pod → NAT Gateway (per AZ) → Internet Gateway → AWS Bedrock |
| Pod → ECR / Secrets Manager / S3 / DynamoDB | Pod → VPC Endpoint (stays in VPC, no NAT cost) |
| Pod → RDS | Direct (same VPC, private subnet) |

**VPC Endpoints (interface):** ECR API, ECR DKR, Secrets Manager, SSM, SSM Messages, EC2 Messages  
**VPC Endpoints (gateway):** S3, DynamoDB

**Security Groups:**
- `alb-sg` — allows 0.0.0.0/0 on 443; forwards to VPC CIDR only
- `eks-cluster-sg` — 443 from NLB; all traffic within the cluster security group
- `vpc-endpoints-sg` — 443 from VPC CIDR
- `rds-sg` — 5432 from EKS node security group only

### Compute (EKS)

**Managed Node Group**
- Instance type: t3.medium (2 vCPU, 4 GB RAM each)
- Capacity type: On-Demand
- EBS: 50 GB per node
- Scaling range: min 2, desired 3–4, max 6 nodes
- AMI: Amazon EKS-optimised (auto-managed by EKS)

**Cluster Autoscaler**
- Deployed via Helm in `kube-system`
- Scales nodes based on pod pending pressure
- Scale-down: 50% utilisation threshold, 10-minute unneeded time, 10-minute delay after add
- Expander policy: `least-waste`

**Workloads (prod namespace):**

| Deployment | Replicas (min–max) | HPA Target |
|---|---|---|
| frontend | 2–10 | CPU 60%, Memory 70% |
| ai-service | 2–5 | CPU 70% |
| travel-service | 2 | — |
| user-service | 2 | — |
| utility-service | 2 | — |

### Database (RDS PostgreSQL)

- **Engine:** PostgreSQL
- **Instance class:** db.t3.micro
- **Storage:** 50 GB, auto-scales to 100 GB
- **Backup retention:** 30 days
- **Encryption:** KMS CMK
- **Network:** Database subnets, no public access
- **Credentials:** Stored in Secrets Manager, injected into pods at runtime via IRSA
- **Schema management:** Alembic migrations run automatically at each pod startup (`alembic upgrade head && gunicorn ...`)

**Databases hosted on single RDS instance:**
- `user_db` — user accounts, authentication
- `travel_db` — trips, expenses

---

## Kubernetes & GitOps

The [`PlanMyJourney-Gitops`](https://github.com/Plan-My-Journey/PlanMyJourney-Gitops) repository is the single source of truth for all Kubernetes state. ArgoCD watches this repo and reconciles the cluster automatically.

### Directory Structure

```
PlanMyJourney-Gitops/
├── helm-charts/                  # Helm chart per service
│   ├── ai-service/
│   │   ├── Chart.yaml
│   │   ├── values.yaml           # Common defaults
│   │   ├── values-dev.yaml       # Dev overrides
│   │   ├── values-prod.yaml      # Prod overrides (HPA, resource limits, ConfigMap)
│   │   └── templates/            # Deployment, Service, HPA, ConfigMap, ServiceMonitor
│   ├── frontend/
│   ├── travel-service/
│   ├── user-service/
│   └── utility-service/
│
├── argocd-apps/
│   ├── app-of-apps.yaml          # Root ArgoCD Application (sync-wave -1)
│   └── applications/
│       ├── infrastructure/       # Gateway, Cluster Autoscaler, Metrics Server, Monitoring
│       ├── dev/                  # Per-service ArgoCD apps for dev namespace
│       └── prod/                 # Per-service ArgoCD apps for prod namespace
│
├── gateway/
│   ├── base/gateway.yaml         # KGateway (Envoy) listener definition
│   └── routes/                   # HTTPRoute per service (URL rewrite + backend ref)
│
├── monitoring/
│   ├── prometheus-rules.yaml
│   ├── grafana-datasources.yaml
│   ├── grafana-route.yaml        # HTTPRoute for Grafana at grafana.invest-iq.online
│   └── kube-prometheus-stack/    # Helm values for kube-prometheus-stack
│
└── scripts/                      # Bootstrap and utility scripts
```

### App-of-Apps Pattern

A single root ArgoCD Application (`planmyjourney-app-of-apps`) recursively syncs all child applications from the `argocd-apps/applications/` directory. Child apps cover infrastructure components and both `dev` and `prod` namespaces.

**Sync configuration (all apps):**
```yaml
syncPolicy:
  automated:
    prune: true      # Removes resources deleted from Git
    selfHeal: true   # Re-applies on manual cluster changes (drift correction)
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
```

**Sync waves (ordering):**
| Wave | Content |
|---|---|
| -1 | Root app-of-apps |
| 0 | KGateway CRDs, gateway controller |
| 1 | Cluster Autoscaler, Metrics Server |
| 2 | Application services (ai-service, frontend, travel-service, etc.) |

### KGateway (Kubernetes Gateway API)

KGateway (Envoy-backed) replaces both ALB and classic Nginx Ingress. HTTPRoute objects define path-based routing rules with URL prefix rewriting (strips the `/api` prefix before forwarding to backend pods).

**Listener:** Port 443, all hostnames, protocol HTTP (TLS terminated upstream at NLB).

**Routes:**
- `invest-iq.online /` → `frontend:8080`
- `api.invest-iq.online /api/auth` → `user-service:8000`
- `api.invest-iq.online /api/trips` → `travel-service:8000`
- `api.invest-iq.online /api/ai` → `ai-service:8000`
- `api.invest-iq.online /api/weather` → `utility-service:8000`
- `grafana.invest-iq.online /` → `monitoring-grafana:80`

### Kubernetes Manifest Organisation

All application manifests are managed as Helm charts in `PlanMyJourney-Gitops/helm-charts/<service>/`. Each chart's `templates/` directory contains:

```
helm-charts/<service>/templates/
├── deployment.yaml        # Deployment with replicas, liveness/readiness probes, resource limits
├── service.yaml           # ClusterIP Service
├── hpa.yaml               # HorizontalPodAutoscaler (CPU-based)
├── configmap.yaml         # Non-sensitive environment variables
├── secret.yaml            # K8s Secret (external-secrets or inline) for sensitive values
├── serviceaccount.yaml    # ServiceAccount with IRSA annotation
├── pdb.yaml               # PodDisruptionBudget
└── servicemonitor.yaml    # Prometheus ServiceMonitor
```

ArgoCD renders these Helm templates and applies the resulting manifests to the cluster (`kubectl apply` equivalent via Helm + ArgoCD Server-Side Apply).

### Secrets Management (K8s Secrets — Not ConfigMaps)

Sensitive values (database URLs, API keys, Cognito credentials) are stored in **Kubernetes Secrets**, not ConfigMaps. The project uses a two-layer approach:

1. **AWS Secrets Manager** stores the actual secret values (encrypted with KMS CMK)
2. **Kubernetes Secrets** are populated from Secrets Manager at deploy time — the Helm chart `secret.yaml` template references the Secrets Manager secret ARN; pods mount the K8s Secret as environment variables

ConfigMaps are used **only** for non-sensitive configuration (log level, feature flags, model IDs, region, queue URLs).

```yaml
# Example: secret.yaml in helm-charts/user-service/templates/
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secret
  namespace: prod
type: Opaque
# Populated by CI from AWS Secrets Manager at deploy time
# Pods reference this Secret via envFrom: [{secretRef: {name: user-service-secret}}]
```

### Health Probes

Every service deployment configures both liveness and readiness probes. FastAPI services expose `/health`, `/healthz`, and `/ready` endpoints on port 8000:

```yaml
# From helm-charts/<service>/templates/deployment.yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

The frontend Nginx container uses `/` as the liveness probe path (returns 200 for static assets).

### RBAC

Kubernetes RBAC is configured for all service accounts. Each service account has the **minimum required permissions**:

- Application service accounts have no cluster-wide RBAC roles — they are restricted to their own namespace via IRSA scoping (AWS resource-level policies limit what each SA can access in AWS)
- The `argocd` service account has read/write access to all namespaces for deployment management
- The `karpenter` service account has cluster-scoped permissions for node lifecycle management
- The `keda-operator` service account has cluster-scoped permissions to manage `ScaledObject` and `HPA` resources
- `metrics-server` has a ClusterRole limited to `nodes/metrics` and `pods/metrics` resources

ArgoCD AppProject definitions (`planmyjourney.yaml`) restrict which source repositories and destination namespaces each application can deploy to, providing an additional RBAC boundary at the GitOps layer.

### Helm Values (prod ai-service)

```yaml
replicaCount: 2

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 1

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"

configMap:
  data:
    BEDROCK_MODEL_ID: "amazon.nova-pro-v1:0"
    BEDROCK_REGION: "us-east-1"
    ENVIRONMENT: "production"
    ASYNC_JOBS_ENABLED: "false"
    BEDROCK_MAX_TOKENS: "4096"
```

---

## CI/CD Pipelines

### Overview

CI/CD is built entirely on GitHub Actions, using 30+ reusable workflows defined in [`PlanMyJourney-Workflows`](https://github.com/Plan-My-Journey/PlanMyJourney-Workflows) and consumed by per-service pipelines in each app repo.

### Three Core Workflows

#### 1. Build Pipeline (`.github/workflows/build.yml`)

Triggered on push to `main` or `develop` and on every Pull Request:

```
Trigger: push to main / PR opened
  ├── Lint & test application code (pytest / eslint)
  ├── SAST scan — SonarCloud static analysis
  ├── SCA scan — Snyk dependency vulnerability scan
  ├── Build Docker image (multi-stage)
  ├── Trivy container scan — fails on HIGH or CRITICAL CVEs
  ├── Push image to ECR with tag: {git-sha}
  └── Post scan results as PR comment
```

Security gate: the pipeline **fails and blocks merge** if Trivy finds any HIGH or CRITICAL vulnerabilities.

#### 2. Deploy Pipeline (`.github/workflows/deploy.yml`)

Triggered on successful completion of the Build Pipeline on `main`:

```
Trigger: build.yml completes successfully on main
  ├── Get EKS cluster credentials (GitHub OIDC → STS → kubeconfig)
  ├── Update image tag in PlanMyJourney-Gitops (values-dev.yaml)
  ├── Trigger ArgoCD Application sync via ArgoCD API
  ├── kubectl rollout status — waits for rolling update to complete
  ├── Smoke tests — health check + basic endpoint validation on deployed pods
  └── Notification — email + GitHub commit status on success or failure
```

Production deployment requires a **manual approval gate** (GitHub Environment protection rule on the `production` environment). No code reaches prod without human review.

#### 3. Infrastructure Pipeline (`.github/workflows/terraform-apply.yml`)

Triggered on changes to the `terraform/` directory:

```
Trigger: push or PR modifying terraform/ files
  ├── terraform init (S3 backend, DynamoDB locking)
  ├── terraform validate
  ├── terraform fmt --check
  ├── terraform plan — output posted as PR comment
  ├── REQUIRE MANUAL APPROVAL (GitHub Environment: infrastructure)
  └── terraform apply — only after approval
```

The plan output is posted as a PR comment so reviewers can see exactly what AWS resources will change before approving. No `terraform apply` runs without explicit human approval.

### Trigger Matrix

| Trigger | Actions |
|---|---|
| Pull Request | SAST (SonarCloud), SCA (Snyk), Docker build + Trivy scan, plan posted as comment |
| Push to `main` | Build → ECR → update GitOps → ArgoCD sync → smoke test → notification |
| Workflow Dispatch (`dev-deploy`) | Manual rebuild and redeploy to dev per service |
| Workflow Dispatch (`release`) | Semver version → ECR tag → git tag → **manual approval** → prod deploy |
| `terraform/` change | Validate → plan → **manual approval** → apply |

### Production Release Pipeline

```
1. Validate semver input (e.g., 1.2.3)
2. Push image to ECR with tag: v1.2.3
3. Create Git tag: {service}/v1.2.3
4. Require manual approval (GitHub Environment gate — "production")
5. Commit v1.2.3 to values-prod.yaml in GitOps repo
6. ArgoCD syncs prod namespace
7. Smoke test prod endpoints
8. Notification on success/failure (email via Brevo + GitHub commit status)
```

### Reusable Workflows

| Workflow | Purpose |
|---|---|
| `_sast.yml` | SonarCloud static analysis |
| `_sca.yml` | Snyk dependency vulnerability scan |
| `_docker-build.yml` | Build image + Trivy scan (no push) |
| `_docker-publish.yml` | Build + push to ECR |
| `_cd.yml` | Update image tag in GitOps repo (git commit) |
| `_argocd-sync.yml` | Trigger ArgoCD Application sync via API |
| `_smoke-test.yml` | Health check + basic endpoint validation |
| `_dev-deploy-backend.yml` | End-to-end dev deployment (build → ECR → GitOps → ArgoCD) |
| `_pr-comment.yml` | Post SAST/SCA/scan results as GitHub PR comment |

### Secrets Used

| Secret | Purpose |
|---|---|
| `AWS_DEPLOY_ROLE_ARN_DEV` | OIDC role for dev deployments |
| `AWS_DEPLOY_ROLE_ARN_PROD` | OIDC role for prod deployments |
| `GITOPS_PAT` | GitHub PAT to commit image tags to GitOps repo |
| `ARGOCD_AUTH_TOKEN` | ArgoCD API authentication |
| `SONAR_TOKEN` | SonarCloud SAST scanning |
| `SNYK_TOKEN` | Snyk dependency scanning |
| `BREVO_API_KEY` | Email notifications on scan failures |

---

## Observability & Monitoring

### In-Cluster Stack (kube-prometheus-stack)

Deployed via ArgoCD from `monitoring/kube-prometheus-stack/` Helm values.

**Components:**
- **Prometheus** — scrapes metrics from all pods every 30 seconds via `ServiceMonitor` objects
- **Grafana** — dashboards for cluster health, pod CPU/memory, HTTP request rates, Bedrock latency
- **AlertManager** — routes alerts to email/Slack
- **Node Exporter** — node-level hardware metrics
- **Kube State Metrics** — Kubernetes object state metrics

**Access:** `https://grafana.invest-iq.online` (admin credentials in `grafana-admin` Secret in `monitoring` namespace)

### CloudWatch (AWS-side)

- **Log groups** (7-day retention in prod): EKS control plane, worker node logs, RDS logs
- **Alarms:** RDS CPU > 80%, RDS disk < 10%, unhealthy targets
- **Notifications:** SNS → email alerts

### FinOps Cost Monitoring

A Lambda function (`cost-anomaly-detector-prod`) runs hourly via EventBridge:
- Reads AWS Cost Explorer data
- Compares against stored baselines (DynamoDB)
- Detects anomalies and budget overruns
- Sends daily cost reports via SES

---

## Security

### Layers of Defence

| Layer | Controls Applied |
|---|---|
| Network | VPC isolation, private subnets, restrictive security groups, no public RDS |
| Transport | NLB TLS termination (ACM), HTTP only inside VPC |
| Identity | IRSA for pods (no static keys), GitHub OIDC for CI, Cognito OAuth for users |
| Secrets | AWS Secrets Manager + KMS CMK, projected OIDC tokens |
| Authentication | JWT (Cognito RS256 or local HS256), rate limiting |
| Supply Chain | SAST (SonarCloud), SCA (Snyk), container scanning (Trivy) |
| Encryption at Rest | RDS (KMS), EBS (KMS), S3 (KMS), ECR (KMS), Secrets Manager (KMS) |
| Audit | CloudTrail (all API calls), AWS Config rules |

### Mandatory Security Checks

| Check | Implementation |
|---|---|
| No hardcoded credentials | No secrets in any `.tf`, `.yaml`, `.py`, or `.env` files committed to Git. `.gitignore` excludes `*.tfstate`, `.env`, `terraform.tfvars`, `credentials`, and `.terraform/` |
| Container runs as non-root | `Dockerfile` declares `USER appuser` (Python services) and `USER nginx` (frontend). `runAsNonRoot: true` set in pod `securityContext` |
| Multistage Docker builds | All 5 Dockerfiles use multi-stage builds: `builder` stage compiles/installs dependencies; `runtime` stage copies only the built artefacts — no build toolchain in production images |
| Container image scanning | Trivy runs on every PR and push; pipeline fails on HIGH or CRITICAL CVEs before the image is pushed to ECR |
| Secrets in K8s Secrets | Database URLs and API keys are stored in **Kubernetes Secrets** (not ConfigMaps). ConfigMaps hold only non-sensitive config (log level, region, model IDs) |
| RBAC configured | Each ServiceAccount has the minimum required permissions. No service account has cluster-admin. Application SAs are namespace-scoped; Karpenter and KEDA SAs have only the cluster permissions they need |
| `.gitignore` complete | Excludes `*.tfstate`, `*.tfstate.backup`, `.terraform/`, `.env`, `terraform.tfvars`, `credentials.*`, `*.pem`, `kubeconfig` |

### Container Security

- All production Dockerfiles use multi-stage builds (minimise image size and attack surface)
- Containers run as non-root users (`nginx` user in frontend, `appuser` in Python services)
- `securityContext.runAsNonRoot: true` and `securityContext.readOnlyRootFilesystem: true` set on all pods
- Trivy scans run on every PR and push; results posted to PR comments; HIGH/CRITICAL blocks the merge
- No secrets baked into images; all injected at runtime via IRSA or Kubernetes Secrets

---

## Local Development

### Prerequisites

- Docker & Docker Compose
- Node.js 20+ (for frontend dev server)
- Python 3.12 (optional, for running services without Docker)

### Setup

```bash
# 1. Clone the app repository
git clone https://github.com/Plan-My-Journey/PlanMyJourney-App.git
cd PlanMyJourney-App

# 2. Copy and configure environment
cp .env.example .env
# Edit .env: set DB passwords, JWT secret, API keys

# 3. Start all backend services + databases
docker compose up --build

# 4. In a separate terminal, start frontend dev server
cd frontend
npm ci
npm run dev
# Open http://localhost:5173
```

### Docker Compose Services

| Service | Port (host) | Description |
|---|---|---|
| `user-db` | 5433 | PostgreSQL for user-service |
| `travel-db` | 5434 | PostgreSQL for travel-service |
| `user-service` | 8011 | Auth & user management |
| `travel-service` | 8012 | Trips & expenses |
| `ai-service` | 8013 | AI itinerary generation |
| `utility-service` | 8014 | Weather, hotels, geocoding |
| `nginx-proxy` | 8080 | Reverse proxy (`/api/*` → services) |
| `frontend` | 5173 | React dev server |

Services depend on their databases being healthy before starting. Alembic migrations run automatically at each backend startup.

### Local API Base URL

```
http://localhost:8080/api
```

### Environment Variables

```env
ENVIRONMENT=local
USER_DB_PASSWORD=<choose-a-strong-password>
TRAVEL_DB_PASSWORD=<choose-a-strong-password>
JWT_SECRET_KEY=<long-random-string-min-32-chars>
CORS_ORIGINS=http://localhost:5173,http://localhost:8080
VITE_API_BASE_URL=http://localhost:8080/api
OPENWEATHER_API_KEY=<your-key>  # optional
GEOAPIFY_API_KEY=<your-key>     # optional
```

---

## Deployment

### Bootstrap (first time)

```bash
# 1. Apply Terraform (from PlanMyjourney-Terraform)
cd environments/
terraform init
terraform plan -var-file=prod.tfvars
terraform apply -var-file=prod.tfvars

# 2. Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name ai-travel-prod

# 3. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 4. Apply root ArgoCD application (bootstraps everything else)
kubectl apply -f argocd-apps/app-of-apps.yaml

# ArgoCD will now sync all infrastructure and application services automatically
```

### Continuous Deployment (ongoing)

Push to `main` in `PlanMyJourney-App` → GitHub Actions builds and pushes the image → updates `values-dev.yaml` in Gitops repo → ArgoCD syncs to `dev` namespace automatically.

For production releases, run the `release` workflow dispatch with a semver version. Requires manual approval in GitHub Actions.

### Rollback

**Via ArgoCD UI or CLI:**
```bash
argocd app rollback prod-ai-service 1   # Roll back to previous revision
```

**Via GitOps (revert the image tag commit):**
```bash
git revert <commit-that-changed-image-tag>
git push origin main
# ArgoCD auto-syncs the previous image tag back to the cluster
```

---

## Scaling

The platform uses three complementary layers of autoscaling to handle variable traffic efficiently.

### KEDA — Event-Driven Pod Autoscaling

[KEDA (Kubernetes Event Driven Autoscaler)](https://keda.sh) drives pod scaling for event-driven workloads based on external queue depth rather than CPU alone.

| Workload | Trigger | Min Pods | Max Pods | Scale-to-Zero |
|---|---|---|---|---|
| `ai-worker` | SQS queue depth (1 pod per 5 messages) | 0 | 10 | Yes — scales to zero when queue is empty |
| `ai-service` | SQS queue depth + HTTP request rate | 2 | 10 | No — always on |

KEDA uses `TriggerAuthentication` backed by IRSA to access SQS queue metrics without static credentials.

### HPA — CPU-Based Pod Autoscaling

Standard Kubernetes HPA handles CPU-driven scaling for stateless services:

| Service | Min Replicas | Max Replicas | CPU Target | Memory Target |
|---|---|---|---|---|
| `frontend` | 2 | 10 | 60% | 70% |
| `travel-service` | 2 | 5 | 70% | — |
| `user-service` | 2 | 5 | 70% | — |
| `utility-service` | 2 | 5 | 70% | — |

### Karpenter — Node Autoscaling

[Karpenter](https://karpenter.sh) provisions and deprovisions EKS nodes just-in-time, faster and more cost-efficiently than the traditional Cluster Autoscaler.

- **NodePool** defines instance families (t3, t3a, m5), architecture (amd64), and capacity types (Spot + On-Demand mix)
- **EC2NodeClass** defines the AMI family, subnet selectors, and security group selectors
- **Consolidation policy** (`WhenUnderutilized`) automatically bin-packs and removes underutilised nodes after 30 seconds
- Nodes are provisioned in under 60 seconds (vs 3–5 minutes with Cluster Autoscaler)
- Spot instances are preferred, with automatic fallback to On-Demand for critical workloads

### Database Scaling

- RDS storage auto-scales (50 GB → 100 GB, no downtime)
- RDS instance class upgrade requires a maintenance window

See [SCALING.md](./SCALING.md) for detailed guidance.

---

## Disaster Recovery

The platform is designed for high availability and rapid recovery from infrastructure failures.

### Recovery Objectives

| Metric | Target | Mechanism |
|---|---|---|
| RTO (Recovery Time Objective) | < 15 minutes | RDS Multi-AZ automatic failover; EKS self-healing pods |
| RPO (Recovery Point Objective) | < 1 minute | RDS Multi-AZ synchronous replication; no data loss on failover |

### RDS High Availability (Multi-AZ)

- **Multi-AZ enabled** — RDS maintains a synchronous standby replica in a second Availability Zone (`us-east-1b`)
- Automatic failover triggers within 60–120 seconds on primary failure with no manual intervention required
- All writes are synchronously committed to both the primary and standby before acknowledging success
- `deletion_protection = true` prevents accidental database deletion
- `skip_final_snapshot = false` — a final snapshot is taken before any planned deletion

### Automated Backups

| Backup Type | Retention | Scope |
|---|---|---|
| RDS automated daily snapshots | 30 days | Full database (user_db + travel_db) |
| RDS Multi-AZ standby | Continuous (synchronous) | Live replication — no RPO gap |
| Terraform state | Indefinite (S3 versioning) | Full infrastructure definition recovery |

### Cross-AZ Resilience

- EKS nodes are distributed across `us-east-1a` and `us-east-1b`
- **Pod Disruption Budgets** ensure at least one pod per service remains available during node failures or voluntary disruptions (Karpenter consolidation, rolling updates)
- NAT Gateways are deployed per-AZ to prevent cross-AZ single points of failure
- RDS database subnets span both AZs

### Infrastructure Recovery

In the event of a full environment loss, the platform can be rebuilt from code:

```bash
# 1. Restore infrastructure from Terraform state (S3 versioned bucket)
terraform init && terraform apply -var-file="environments/prod.tfvars"

# 2. Bootstrap ArgoCD — automatically re-deploys all services via GitOps
kubectl apply -f argocd-apps/app-of-apps.yaml

# 3. Restore database from latest RDS snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier ai-travel-prod-restored \
  --db-snapshot-identifier <snapshot-id> \
  --region us-east-1
```

### Backup Verification

Database restore drills are conducted quarterly to validate that backups are restorable and that the restored database passes application-level health checks.

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full DR architecture.

---

## Cost Management (FinOps)

- **VPC Endpoints** for ECR, S3, Secrets Manager, DynamoDB — traffic stays in-VPC, reducing NAT Gateway data processing charges
- **Cluster Autoscaler** — nodes scale down during low-traffic periods
- **Spot / Reserved:** Currently On-Demand; reserved instances recommended for baseline node count
- **RDS:** Single-AZ in production (multi-AZ upgrade available in maintenance window)
- **CloudWatch retention:** 7 days in prod (reduces log storage cost)
- **FinOps Lambda** — hourly cost monitoring, daily email reports, anomaly detection

See [FINOPS.md](./FINOPS.md) for the full cost breakdown and optimisation recommendations.

---

## API Documentation

Every FastAPI service automatically generates interactive OpenAPI (Swagger) documentation. The docs are available at runtime via the standard FastAPI endpoints:

| Service | Swagger UI | ReDoc | OpenAPI JSON |
|---|---|---|---|
| user-service | `/docs` | `/redoc` | `/openapi.json` |
| travel-service | `/docs` | `/redoc` | `/openapi.json` |
| ai-service | `/docs` | `/redoc` | `/openapi.json` |
| utility-service | `/docs` | `/redoc` | `/openapi.json` |

In production these endpoints are internal (not exposed via KGateway's public routes). They are accessible during development via `kubectl port-forward`:

```bash
# Access ai-service Swagger UI locally
kubectl port-forward svc/ai-service 8003:8000 -n prod
# Open: http://localhost:8003/docs
```

All request and response schemas are documented via Pydantic models, which FastAPI uses to generate the OpenAPI specification automatically. Every endpoint includes:
- Input schema with field types and validation rules
- Response schema with example payloads
- Authentication requirement (JWT Bearer token)
- HTTP status codes and error responses

---

## Detailed Documentation

| Document | Contents |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Detailed system architecture diagrams and component interactions |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | Step-by-step deployment procedures for dev and prod |
| [GITOPS.md](./GITOPS.md) | GitOps workflow, ArgoCD configuration, sync waves, and rollback procedures |
| [SCALING.md](./SCALING.md) | KEDA event-driven scaling, HPA configuration, Karpenter node provisioning, database scaling |
| [FINOPS.md](./FINOPS.md) | Cost architecture, VPC endpoint savings, and FinOps Lambda setup |
| [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) | Common issues, debugging commands, and runbooks |

---

## Author

**Preethi K Gowda**  
GitHub: [@Preethikgowda](https://github.com/Preethikgowda)
