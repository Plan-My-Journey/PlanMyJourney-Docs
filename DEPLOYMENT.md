# AI-Travel-Planner: Deployment Guide

## Overview

This guide covers the complete end-to-end deployment of the AI-Travel-Planner platform to AWS. Deployment
is split into two distinct phases:

- **Phase A – Infrastructure**: Terraform provisions the foundational AWS resources (VPC, EKS, RDS,
  ECR, Secrets Manager, FinOps Lambda, etc.). This phase runs once and is idempotent.
- **Phase B – Applications**: Kubernetes workloads are deployed via ArgoCD using a GitOps app-of-apps
  pattern. Image builds are automated through GitHub Actions CI/CD pipelines.

Total estimated first-time deployment time: **40–60 minutes** (infrastructure ~25–35 min, apps ~10–15 min).

---

## Prerequisites

Ensure all tools are installed and authenticated **before** starting.

| Tool | Version | Installation Link | Purpose |
|------|---------|-------------------|---------|
| AWS CLI | v2.x | https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html | Authenticates with AWS; ECR login |
| Terraform | >= 1.7 | https://developer.hashicorp.com/terraform/install | Provisions all AWS infrastructure |
| kubectl | >= 1.29 | https://kubernetes.io/docs/tasks/tools/ | Manages Kubernetes resources |
| Helm | >= 3.14 | https://helm.sh/docs/intro/install/ | Packages and installs Kubernetes apps |
| ArgoCD CLI | >= 2.9 | https://argo-cd.readthedocs.io/en/stable/cli_installation/ | Manages GitOps sync and rollbacks |
| Docker | >= 24.x | https://docs.docker.com/engine/install/ | Builds and pushes container images |
| jq | any | https://jqlang.github.io/jq/download/ | Parses JSON in shell scripts |
| yq | >= 4.x | https://github.com/mikefarah/yq | Parses/edits YAML in shell scripts |
| PowerShell | >= 7.x | https://aka.ms/powershell | Required for `.ps1` build scripts (Windows) |

### Verify All Tools

```bash
aws --version           # aws-cli/2.x.x
terraform version       # Terraform v1.7.x
kubectl version --client # v1.29.x
helm version            # v3.14.x
argocd version          # argocd: v2.9.x
docker --version        # Docker version 24.x.x
jq --version            # jq-1.x
yq --version            # yq version v4.x.x
```

---

## Phase A: Infrastructure Deployment

### 1. AWS Account Setup

#### Configure AWS CLI Profile

```bash
aws configure --profile ai-travel-prod
# AWS Access Key ID: <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region name: ap-south-1
# Default output format: json

# Verify authentication
aws sts get-caller-identity --profile ai-travel-prod
```

#### Required IAM Permissions

The deploying IAM user/role must have broad permissions. The minimum required managed policies are:

| Policy | Reason |
|--------|--------|
| `AdministratorAccess` | Recommended for initial setup |
| OR custom policy covering: | |
| `AmazonEKSFullAccess` | EKS cluster creation |
| `AmazonEC2FullAccess` | VPC, subnets, security groups, NAT |
| `AmazonRDSFullAccess` | RDS instance and parameter groups |
| `AmazonS3FullAccess` | Terraform state bucket, ALB logs |
| `AWSLambda_FullAccess` | FinOps Lambda function |
| `AmazonDynamoDBFullAccess` | Terraform state lock table |
| `SecretsManagerReadWrite` | Application secrets |
| `AmazonBedrockFullAccess` | Bedrock model access |
| `IAMFullAccess` | IRSA roles for pods |

#### Enable Required AWS Services

Some services must be explicitly enabled before Terraform can use them:

```bash
# Enable Cost Explorer (first-time: can take up to 24h to activate)
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Active

# Verify Bedrock is available in ap-south-1
aws bedrock list-foundation-models \
  --region ap-south-1 \
  --query "modelSummaries[?modelId=='amazon.nova-pro-v1:0']"
```

---

### 2. Enable Amazon Bedrock Model Access

The AI service (`ai-service`) uses `amazon.nova-pro-v1:0` via the Bedrock Converse API. Model access
must be **manually requested** before Terraform deploys the IRSA permissions.

**Steps:**

1. Open the [AWS Console](https://console.aws.amazon.com) in region **ap-south-1** (Mumbai)
2. Navigate to **Amazon Bedrock** → **Model access** (left sidebar)
3. Click **Manage model access**
4. Find **Amazon Nova Pro** (`amazon.nova-pro-v1:0`) and check the box
5. Click **Request model access** → **Submit**
6. Wait for status to change from `Available to request` → **`Access granted`** (usually instant for Nova Pro)

> **Important:** Without this step, the `ai-service` pod will start successfully but return fallback
> responses. Check the pod logs for `AccessDeniedException` errors from Bedrock.

---

### 3. Bootstrap Terraform State Backend

The state backend (S3 + DynamoDB) must exist before running `terraform init`. A dedicated bootstrap
script handles this:

```bash
cd infrastructure/terraform
make bootstrap
```

**What `make bootstrap` creates:**

| Resource | Name | Purpose |
|----------|------|---------|
| S3 Bucket | `ai-travel-terraform-state-{account_id}` | Stores `.tfstate` files with versioning + encryption |
| DynamoDB Table | `ai-travel-terraform-locks` | Prevents concurrent `terraform apply` runs (state locking) |
| S3 Bucket Policy | (attached to above) | Enforces HTTPS-only access |
| KMS Key | (CMK) | Encrypts all state files at rest |

If you need to run this manually:

```bash
# Bootstrap is idempotent — safe to run multiple times
cd infrastructure/terraform
bash scripts/bootstrap-state.sh

# Verify S3 bucket was created
aws s3 ls | grep ai-travel-terraform-state
```

---

### 4. Deploy Infrastructure with Terraform

```bash
cd infrastructure/terraform

# Initialize: downloads providers, configures backend
make init
# OR: terraform init -backend-config="environments/prod-backend.hcl"

# Plan: preview all resources to be created (no changes made)
make plan
# OR: terraform plan -var-file="environments/prod.tfvars" -out=tfplan

# !!! REVIEW THE PLAN OUTPUT CAREFULLY !!!
# Look for: resource counts, any unexpected destroys (~)
# Expected: ~120 resources to add, 0 to change, 0 to destroy

# Apply: create all resources (~25–35 minutes)
make apply
# OR: terraform apply tfplan
```

**Resources created during `make apply`:**

| Category | Resources |
|----------|-----------|
| Networking | VPC, 6 subnets (2 public, 2 private, 2 DB), IGW, 2 NAT GWs, route tables, NACLs |
| EKS | Cluster (v1.29), managed node group (t3.medium × 2), OIDC provider |
| Add-ons | AWS VPC CNI, CoreDNS, kube-proxy, EBS CSI driver |
| RDS | PostgreSQL 15.4 Multi-AZ instance (db.t3.micro), subnet group, parameter group |
| ECR | 5 repositories: frontend, user-service, travel-service, ai-service, utility-service |
| IAM | 8 IRSA roles (per-service + cluster autoscaler + ALB controller + FinOps) |
| Secrets | 6 secrets in Secrets Manager (GitHub token, DB credentials, API keys) |
| FinOps | Lambda function, EventBridge rules, SNS topic, DynamoDB table |
| Observability | CloudWatch log groups, metric alarms, dashboard |
| VPC Endpoints | S3 (gateway), DynamoDB (gateway), ECR API, ECR DKR, Secrets Manager, SSM, Bedrock |

---

### 5. Collect Terraform Outputs

After `make apply` completes, capture output values needed for Phase B:

```bash
cd infrastructure/terraform
terraform output

# Save to a file for reference
terraform output -json > /tmp/tf-outputs.json

# Key values to note:
terraform output cluster_name          # ai-travel-prod
terraform output cluster_endpoint      # https://XXXX.gr7.ap-south-1.eks.amazonaws.com
terraform output rds_endpoint          # ai-travel-prod.XXXX.ap-south-1.rds.amazonaws.com:5432
terraform output ecr_repository_urls   # map of service → ECR URL
terraform output alb_dns_name          # ai-travel-XXXX.ap-south-1.elb.amazonaws.com
terraform output finops_lambda_arn     # arn:aws:lambda:ap-south-1:XXXX:function:...
terraform output vpc_id                # vpc-XXXXXXXXXX
```

---

### 6. Populate Secrets in AWS Secrets Manager

Several secrets must be populated manually after infrastructure creation. Terraform creates the
secret shells; this step fills in the actual values.

```bash
# GitHub Personal Access Token (for ArgoCD to pull from private repos)
aws secretsmanager put-secret-value \
  --secret-id "ai-travel/github-token" \
  --secret-string '{"token": "ghp_XXXXXXXXXXXX"}' \
  --region ap-south-1

# Third-party API keys for utility-service (OpenWeather, Amadeus, etc.)
aws secretsmanager put-secret-value \
  --secret-id "ai-travel/utility-api-keys" \
  --secret-string '{
    "OPENWEATHER_API_KEY": "your-openweather-key",
    "AMADEUS_CLIENT_ID": "your-amadeus-client-id",
    "AMADEUS_CLIENT_SECRET": "your-amadeus-client-secret",
    "CURRENCY_API_KEY": "your-currency-api-key"
  }' \
  --region ap-south-1

# FinOps email configuration (for cost alert emails)
aws secretsmanager put-secret-value \
  --secret-id "ai-travel/finops-config" \
  --secret-string '{
    "NOTIFICATION_EMAIL": "devops@yourcompany.com",
    "COST_ALERT_THRESHOLD": "500"
  }' \
  --region ap-south-1

# Verify secrets are populated
aws secretsmanager list-secrets \
  --region ap-south-1 \
  --query "SecretList[?starts_with(Name, 'ai-travel')].{Name: Name, LastChanged: LastChangedDate}"
```

> **Note:** The `DATABASE_URL` secret is auto-populated by Terraform using the RDS endpoint and
> credentials generated during `make apply`. You do not need to set this manually.

> **Bedrock:** No API key is required. The `ai-service` uses IRSA to authenticate with Bedrock
> via the pod's service account token. The GROQ dependency has been removed.

---

## Phase B: Application Deployment

### 1. Configure kubectl for EKS

```bash
# Update local kubeconfig (replaces or merges with ~/.kube/config)
aws eks update-kubeconfig \
  --name ai-travel-prod \
  --region ap-south-1 \
  --profile ai-travel-prod

# Verify cluster access
kubectl get nodes
# Expected output (after ~2 min):
# NAME                                       STATUS   ROLES    AGE   VERSION
# ip-10-0-10-xx.ap-south-1.compute.internal  Ready    <none>   2m    v1.29.x

# Check all system pods are running
kubectl get pods -n kube-system
```

---

### 2. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD (stable release)
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all ArgoCD components to become ready
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=300s

kubectl wait --for=condition=available deployment/argocd-application-controller \
  -n argocd --timeout=300s

# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d ; echo

# Access ArgoCD UI via port-forward (temporary)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080  (user: admin, password: <from above>)
```

> **Production setup:** Expose ArgoCD via the ALB ingress (see `infrastructure/argocd-apps/argocd-ingress.yaml`)
> so it is accessible via `https://argocd.aitravel.com` without port-forwarding.

---

### 3. Bootstrap App-of-Apps

```bash
# Create the ArgoCD project (defines permissions and allowed repos)
kubectl apply -f infrastructure/argocd-apps/projects/ai-travel.yaml

# Apply the root app-of-apps (this single manifest deploys ALL services)
kubectl apply -f infrastructure/argocd-apps/app-of-apps.yaml

# Watch ArgoCD sync all child applications (~5–8 minutes)
kubectl get applications -n argocd -w
# Expected: All apps reach Status=Synced Health=Healthy

# OR use ArgoCD CLI
argocd login localhost:8080 --username admin --password <password> --insecure
argocd app list
```

---

### 4. Build and Push Docker Images

```bash
# Get AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="ap-south-1"
ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

# Authenticate Docker with ECR
aws ecr get-login-password --region ${REGION} | \
  docker login --username AWS --password-stdin ${ECR_REGISTRY}

# Build and push all services (Windows PowerShell)
./scripts/build-and-push-ecr.ps1 -Region ${REGION} -AccountId ${ACCOUNT_ID}

# OR build individual services manually:
# Frontend
docker build -t ${ECR_REGISTRY}/ai-travel/frontend:latest \
  -f frontend/Dockerfile frontend/
docker push ${ECR_REGISTRY}/ai-travel/frontend:latest

# User Service
docker build -t ${ECR_REGISTRY}/ai-travel/user-service:latest \
  -f services/user-service/Dockerfile services/user-service/
docker push ${ECR_REGISTRY}/ai-travel/user-service:latest

# AI Service
docker build -t ${ECR_REGISTRY}/ai-travel/ai-service:latest \
  -f services/ai-service/Dockerfile services/ai-service/
docker push ${ECR_REGISTRY}/ai-travel/ai-service:latest
```

---

### 5. Verify Deployment

```bash
# Check all applications synced in ArgoCD
argocd app list
# All apps should show: STATUS=Synced  HEALTH=Healthy

# Check all pods are running in the ai-travel namespace
kubectl get pods -n ai-travel
# Expected: frontend, user-service, travel-service, ai-service, utility-service all Running

# Check ingress was created and has an ADDRESS
kubectl get ingress -n ai-travel
# NAME       CLASS    HOSTS                ADDRESS                          PORTS
# frontend   nginx    aitravel.com         ai-travel-XXXX.elb.amazonaws.com 80,443

# Smoke test: health check each service
curl https://api.aitravel.com/health          # should return 200
curl https://api.aitravel.com/users/health    # 200
curl https://api.aitravel.com/travel/health   # 200
curl https://api.aitravel.com/ai/health       # 200
curl https://api.aitravel.com/utility/health  # 200
```

---

### 6. DNS Configuration

```bash
# Get ALB DNS name from Terraform output or kubectl
ALB_DNS=$(kubectl get ingress frontend -n ai-travel \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo $ALB_DNS  # ai-travel-XXXX.ap-south-1.elb.amazonaws.com

# Option A: Route53 (recommended)
# Create CNAME or ALIAS record in your hosted zone:
#   aitravel.com     → ALIAS → ${ALB_DNS}
#   www.aitravel.com → CNAME → ${ALB_DNS}

# Option B: External DNS (automated, if installed)
# Add annotation to ingress:
#   external-dns.alpha.kubernetes.io/hostname: aitravel.com

# Verify DNS propagation
dig aitravel.com         # should resolve to ALB IP
curl -I https://aitravel.com   # should return HTTP 200
```

---

## Post-Deployment Checklist

- [ ] `kubectl get nodes` shows all nodes `Ready`
- [ ] `kubectl get pods -n ai-travel` shows all pods `Running` (no restarts)
- [ ] `argocd app list` shows all apps `Synced` and `Healthy`
- [ ] Health endpoint returns 200 for all 5 services
- [ ] ArgoCD UI accessible (https://argocd.aitravel.com or via port-forward)
- [ ] Bedrock model access enabled (`amazon.nova-pro-v1:0` in ap-south-1)
- [ ] DNS resolves to ALB correctly
- [ ] HTTPS certificate is valid (check via browser or `curl -I`)
- [ ] All secrets populated in Secrets Manager (verify via AWS Console)
- [ ] FinOps Lambda executes without error (check CloudWatch Logs)
- [ ] CloudWatch alarms are in `OK` state (no immediate alerts)
- [ ] RDS accessible from within EKS pods (test with `kubectl exec`)
- [ ] ECR image scan shows no critical vulnerabilities
- [ ] GitHub Actions CI pipeline passes on a test commit

---

## Teardown (Destroy All Resources)

> **Warning:** This permanently destroys all data including RDS databases. Take manual snapshots first.

```bash
# Step 1: Delete all Kubernetes resources (ArgoCD cascade delete)
argocd app delete app-of-apps --cascade

# Step 2: Wait for namespace cleanup
kubectl wait --for=delete namespace/ai-travel --timeout=120s

# Step 3: Destroy Terraform infrastructure
cd infrastructure/terraform
make destroy
# OR: terraform destroy -var-file="environments/prod.tfvars"

# Step 4: (Optional) Remove Terraform state backend
# Only if you want to fully clean up the AWS account
aws s3 rb s3://ai-travel-terraform-state-${ACCOUNT_ID} --force
aws dynamodb delete-table --table-name ai-travel-terraform-locks --region ap-south-1
```
