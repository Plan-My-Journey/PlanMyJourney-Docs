# FinOps Operations Guide

## Overview

The FinOps system provides **automated cost monitoring, anomaly detection, and AI-powered
optimization recommendations** for the AI-Travel-Planner infrastructure. It runs hourly via AWS
EventBridge and produces structured cost reports stored in DynamoDB.

Key capabilities:
- **Daily cost analysis**: MTD spend, daily trend, per-service breakdown
- **Anomaly detection**: Alerts when costs spike >15% vs 7-day rolling average
- **AI recommendations**: Uses Bedrock Nova Pro to analyze cost patterns and suggest optimizations
- **Rightsizing**: Integrates with AWS Compute Optimizer for EC2/RDS instance recommendations
- **Tag compliance**: Scans all resources for required cost allocation tags
- **GitHub PRs**: Generates pull requests with Terraform changes for approved rightsizing actions
- **Real-time alerts**: EventBridge rules fire SNS notifications on unexpected resource creation

---

## Architecture

```
╔══════════════════════════════════════════════════════════════════╗
║                    FinOps Architecture                           ║
╚══════════════════════════════════════════════════════════════════╝

EventBridge Schedule (hourly — `rate(1 hour)`, rule: `cost-anomaly-detector-prod-daily`)
          │
          ▼
  ┌─────────────────────────────────────────────────────┐
  │     Lambda: ai-travel-finops-cost-analyzer          │
  │     Runtime: Python 3.12  |  Memory: 512 MB         │
  │     Timeout: 5 minutes                              │
  │                                                     │
  │  Step 1: Cost Explorer API                          │
  │    ├── Get MTD total cost                           │
  │    ├── Get daily cost for last 30 days              │
  │    ├── Get cost by AWS service                      │
  │    └── Get cost by cost allocation tag (Project)    │
  │                                                     │
  │  Step 2: Compute Optimizer API                      │
  │    ├── EC2 instance recommendations                 │
  │    └── EBS volume recommendations                   │
  │                                                     │
  │  Step 3: CloudWatch Metrics                         │
  │    ├── EKS node CPU/memory utilization (7-day avg)  │
  │    └── RDS CPU/connections/storage (7-day avg)      │
  │                                                     │
  │  Step 4: ResourceGroupsTaggingAPI                   │
  │    └── Scan all resources for tag compliance        │
  │                                                     │
  │  Step 5: Bedrock Nova Pro (Converse API)            │
  │    ├── Input: cost data + metrics + recommendations │
  │    └── Output: natural-language analysis + actions  │
  │                                                     │
  │  Step 6: Persist Results                            │
  │    ├── DynamoDB: store full report (90-day TTL)     │
  │    ├── SNS: alert if anomaly detected               │
  │    └── SES: send daily/weekly email digest          │
  └─────────────────────────────────────────────────────┘

EventBridge Real-Time Rules (always active):
  ├── EC2 RunInstances event    → SNS alert within 60 seconds
  ├── RDS CreateDB/ModifyDB     → SNS alert within 60 seconds
  └── IAM CreateUser/AttachPolicy → SNS alert within 60 seconds
```

### Data Flow for AI Analysis

```
Cost Explorer (raw $) ──┐
Compute Optimizer ──────┤
CloudWatch Metrics ─────┼──► Structured JSON ──► Bedrock Nova Pro ──► Recommendations
Tag Compliance ─────────┘                         (Converse API)           │
                                                                            ▼
                                                                    DynamoDB report
                                                                    SNS/SES alerts
                                                                    GitHub PR (opt)
```

---

## Cost Alert Thresholds

| Alert Type | Default Threshold | Config Variable | Action |
|------------|------------------|----------------|--------|
| MTD cost limit | $500 | `COST_ALERT_THRESHOLD` | SNS → email |
| Daily anomaly | >15% vs 7-day avg | `ANOMALY_PCT_THRESHOLD` | SNS → email |
| Single service spike | >50% day-over-day | `SERVICE_SPIKE_THRESHOLD` | SNS → email |
| Tag compliance | <90% compliant | `TAG_COMPLIANCE_MIN_PCT` | SNS → weekly digest |
| RDS storage | <20% free space | (CloudWatch alarm) | SNS → PagerDuty |

### Customizing Thresholds

Thresholds are configurable via Lambda environment variables. Update via Terraform:

```hcl
# infrastructure/terraform/modules/finops/lambda.tf
resource "aws_lambda_function" "cost_analyzer" {
  environment {
    variables = {
      COST_ALERT_THRESHOLD     = "750"    # Increase limit to $750
      ANOMALY_PCT_THRESHOLD    = "20"     # Alert only on >20% spikes
      SERVICE_SPIKE_THRESHOLD  = "60"     # Alert on >60% service spike
      TAG_COMPLIANCE_MIN_PCT   = "95"     # Stricter tag compliance
    }
  }
}
```

---

## Email Notifications

### Notification Schedule

| Notification | Trigger | Recipients |
|-------------|---------|-----------|
| Cost anomaly alert | MTD exceeds threshold OR daily anomaly detected | SNS → `tkpreethi973@gmail.com` |
| Daily cost report | Every Lambda run (hourly) | SES from `tkp4762@gmail.com` → `tkpreethi973@gmail.com` |
| Weekly digest | When weekly rule is configured | SES from `tkp4762@gmail.com` → `tkpreethi973@gmail.com` |
| Real-time alert | EC2/RDS/IAM change via EventBridge | SNS → `alert_email` in Terraform |
| Rightsizing PR | When Compute Optimizer has high-confidence recommendations | GitHub PR (no email) |

### Sample Email: Cost Anomaly Alert

```
Subject: [AI-Travel] 🚨 Cost Anomaly Detected — $312 MTD (up 23% vs avg)

AI Travel Planner — Daily Cost Report
Date: 2026-06-18 | Region: ap-south-1

ANOMALY DETECTED
  Current MTD:  $312.40
  7-day avg MTD: $254.20
  Variance:      +$58.20 (+22.9%)

Top cost drivers today:
  1. NAT Gateway:     $8.40  (+41% vs avg) ← investigate data transfer
  2. RDS:             $5.90  (+12% vs avg)
  3. EKS nodes:       $4.80  (normal)

AI Recommendation (powered by Bedrock):
  "The NAT Gateway cost spike suggests unusual outbound data transfer.
   Review CloudWatch VPC flow logs for the source pod or service generating
   unexpected traffic. Consider enabling S3/ECR VPC endpoints if not already
   active to reduce NAT costs for AWS API calls."

Actions:
  [View Full Report] https://console.aws.amazon.com/dynamodb/...
  [View Cost Explorer] https://console.aws.amazon.com/cost-management/...
```

### Sample Email: Weekly Digest

```
Subject: [AI-Travel] Weekly Cost Digest — Week of Jun 12–18, 2026

7-Day Summary:
  Total spend:        $79.20
  Budget:             $80.00 (99% utilized this week)
  Month-to-date:      $231.40
  Projected monthly:  $318.00

Top Services:
  EKS (control plane + nodes): $24.80  (31%)
  RDS Multi-AZ:                $14.70  (19%)
  NAT Gateways:                $17.20  (22%)
  VPC Endpoints:               $11.20  (14%)
  Other:                       $11.30  (14%)

Optimization Opportunities (from Compute Optimizer):
  • RDS: Consider Graviton db.t4g.micro (-15% cost, similar performance)
  • EC2: Reserved Instance purchase for t3.medium saves ~$14/month

No anomalies detected this week. System healthy.
```

---

## Reviewing Cost Recommendations

### Step 1: Read the DynamoDB Report

```bash
# Get the latest FinOps report
aws dynamodb scan \
  --table-name ai-travel-finops-reports \
  --filter-expression "#ts > :cutoff" \
  --expression-attribute-names '{"#ts": "timestamp"}' \
  --expression-attribute-values '{":cutoff": {"S": "2026-06-17"}}' \
  --region ap-south-1 \
  --output json | jq '.Items[] | {timestamp: .timestamp.S, summary: .ai_summary.S}'
```

### Step 2: Read Rightsizing Recommendations

```bash
# Get Compute Optimizer EC2 recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --region ap-south-1 \
  --query "instanceRecommendations[].{
    Instance: instanceArn,
    CurrentType: currentInstanceType,
    RecommendedType: recommendationOptions[0].instanceType,
    EstimatedSavings: recommendationOptions[0].estimatedMonthlySavings.value
  }" \
  --output table

# Get RDS recommendations
aws compute-optimizer get-rds-database-recommendations \
  --region ap-south-1 \
  --output table
```

### Step 3: Review CloudWatch Dashboards

1. Open [CloudWatch Console](https://console.aws.amazon.com/cloudwatch) in `ap-south-1`
2. Navigate to **Dashboards** → **ai-travel-finops**
3. Review:
   - **Cost trend** (30-day line chart)
   - **Service breakdown** (pie chart by AWS service)
   - **EKS node utilization** (CPU %, memory %)
   - **RDS metrics** (CPU, connections, storage free)
4. If EKS CPU is consistently <30%, consider scaling down or switching to t3.small

### Step 4: Run FinOps Lambda Manually

```bash
# Trigger on-demand run (no need to wait for scheduled event)
aws lambda invoke \
  --function-name ai-travel-finops-cost-analyzer \
  --region ap-south-1 \
  --log-type Tail \
  --output json \
  response.json

# Check Lambda logs
cat response.json | jq .
aws logs tail /aws/lambda/ai-travel-finops-prod \
  --since 10m \
  --region ap-south-1
```

---

## Acting on Rightsizing Recommendations

When Compute Optimizer identifies a high-confidence rightsizing opportunity, the FinOps Lambda
can create a GitHub PR with the Terraform changes. Review and merge the PR to apply the change.

### Manual Rightsizing: RDS Instance Type

```bash
# 1. Update Terraform variable
# infrastructure/terraform/environments/prod.tfvars
rds_instance_class = "db.t3.small"   # was db.t3.micro

# 2. Plan the change
cd infrastructure/terraform
terraform plan -var-file="environments/prod.tfvars" -target=module.rds

# 3. Apply during maintenance window (causes ~20s failover for Multi-AZ)
terraform apply -var-file="environments/prod.tfvars" -target=module.rds

# 4. Verify
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --query "DBInstances[0].{Class: DBInstanceClass, Status: DBInstanceStatus}"
```

### Manual Rightsizing: EKS Node Type

```bash
# 1. Update terraform variables
# prod.tfvars
node_instance_types     = ["t3.large"]   # was t3.medium
node_group_desired_size = 2

# 2. Plan (will show managed node group update)
terraform plan -var-file="environments/prod.tfvars" -target=module.eks

# 3. Apply (triggers rolling node replacement — no downtime with 2+ nodes)
terraform apply -var-file="environments/prod.tfvars" -target=module.eks

# 4. Verify new node type
kubectl get nodes -o custom-columns=NAME:.metadata.name,INSTANCE:.metadata.labels.'beta\.kubernetes\.io/instance-type'
```

---

## Cost Optimization Playbook

### 1. EC2 / EKS Node Rightsizing

| Trigger | Investigation | Action |
|---------|--------------|--------|
| CPU consistently < 20% for 7 days | `kubectl top nodes`, CloudWatch EKS metrics | Reduce node instance size or count |
| CPU consistently > 70% for 7 days | `kubectl top pods -n ai-travel` | Increase node size or count |
| Memory OOMKilled | `kubectl describe node` events | Increase limits or upgrade instance |

```bash
# Check current node utilization
kubectl top nodes

# Check pod resource utilization
kubectl top pods -n ai-travel --sort-by=memory
```

### 2. Reserved Instance Purchase Process

Reserved Instances reduce EKS node and RDS costs by 30–40% for 1-year commitments.

```bash
# 1. Confirm you have stable, consistent usage (use Cost Explorer)
aws ce get-reservation-purchase-recommendation \
  --service "Amazon Elastic Compute Cloud - Compute" \
  --region ap-south-1 \
  --output table

# 2. Purchase via Console or CLI
# AWS Console → EC2 → Reserved Instances → Purchase Reserved Instances
# Or RDS Console → Reserved DB Instances → Purchase Reserved DB Instance

# Recommended purchases for this setup:
# - 1 × t3.medium Linux on-demand → 1-year no-upfront RI: saves ~$14/month
# - 1 × db.t3.micro Multi-AZ → 1-year no-upfront RI: saves ~$22/month
```

### 3. NAT Gateway Cost Reduction

NAT Gateway data processing ($0.045/GB) is often the largest variable cost. Strategies:

```bash
# Check NAT Gateway data usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=<nat-gw-id> \
  --start-time 2026-06-11T00:00:00Z \
  --end-time 2026-06-18T00:00:00Z \
  --period 86400 \
  --statistics Sum \
  --region ap-south-1

# Reduction strategies:
# 1. Verify all AWS API calls use VPC endpoints (already configured)
# 2. Check for pods making excessive external API calls
#    kubectl exec -it <pod> -n ai-travel -- curl -v https://external-api.com
# 3. Enable S3 Transfer Acceleration for large uploads (avoids regional transfer)
```

### 4. S3 Lifecycle Rule Review

```bash
# Review existing lifecycle rules
aws s3api get-bucket-lifecycle-configuration \
  --bucket ai-travel-terraform-state-<account-id>

# Add lifecycle rule to move old state versions to Glacier after 90 days
aws s3api put-bucket-lifecycle-configuration \
  --bucket ai-travel-terraform-state-<account-id> \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "archive-old-state-versions",
      "Status": "Enabled",
      "NoncurrentVersionTransitions": [{
        "NoncurrentDays": 90,
        "StorageClass": "GLACIER"
      }]
    }]
  }'
```

### 5. Unused Resource Cleanup

```bash
# Find unattached EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --region ap-south-1 \
  --query "Volumes[].{VolumeId: VolumeId, Size: Size, Created: CreateTime}" \
  --output table

# Find old ECR images (keep only last 10 per repo)
aws ecr list-images \
  --repository-name ai-travel/user-service \
  --region ap-south-1 \
  --filter tagStatus=UNTAGGED \
  --query "imageIds[].imageDigest" \
  --output text

# Enable ECR lifecycle policy to auto-delete old images
aws ecr put-lifecycle-policy \
  --repository-name ai-travel/user-service \
  --region ap-south-1 \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "description": "Keep last 20 tagged images",
      "selection": {"tagStatus": "tagged", "tagPrefixList": ["sha-"], "countType": "imageCountMoreThan", "countNumber": 20},
      "action": {"type": "expire"}
    }]
  }'
```

---

## FinOps Lambda Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `NOTIFICATION_EMAIL` | Email address for cost alerts | (from Secrets Manager) | Yes |
| `COST_ALERT_THRESHOLD` | MTD cost alert threshold in USD | `500` | No |
| `ANOMALY_PCT_THRESHOLD` | % increase to trigger anomaly alert | `15` | No |
| `SERVICE_SPIKE_THRESHOLD` | % day-over-day increase per service | `50` | No |
| `TAG_COMPLIANCE_MIN_PCT` | Minimum % of resources with required tags | `90` | No |
| `AWS_REGION` | AWS region for all API calls | `ap-south-1` | Yes |
| `DYNAMODB_TABLE` | DynamoDB table for report storage | `ai-travel-finops-reports` | Yes |
| `SNS_TOPIC_ARN` | SNS topic for anomaly alerts | (Terraform output) | Yes |
| `BEDROCK_MODEL_ID` | Bedrock model for AI analysis | `amazon.nova-pro-v1:0` | No |
| `REPORT_RETENTION_DAYS` | DynamoDB TTL for reports | `90` | No |
| `GITHUB_REPO` | Repo for rightsizing PRs (optional) | `` | No |
| `GITHUB_TOKEN_SECRET` | Secrets Manager secret name for GitHub token | `ai-travel/github-token` | No |
| `ENABLE_RIGHTSIZING_PRS` | Create GitHub PRs for rightsizing | `false` | No |
| `WEEKLY_DIGEST_DAY` | Day of week for weekly digest (0=Mon) | `0` | No |
| `LOG_LEVEL` | Lambda log verbosity | `INFO` | No |

---

## Adding Custom Cost Checks

The FinOps Lambda is designed to be extensible. Add a new analyzer by creating a Python function
and registering it in the analyzer chain.

### Example: Add an Unused Elastic IP Checker

```python
# infrastructure/terraform/modules/finops/lambda/analyzers/unused_eips.py
import boto3
from typing import Dict, Any

def analyze_unused_eips(session: boto3.Session) -> Dict[str, Any]:
    """
    Check for Elastic IPs that are allocated but not associated
    with any EC2 instance (wasteful — $3.60/month each).
    """
    ec2 = session.client('ec2')
    response = ec2.describe_addresses(
        Filters=[{'Name': 'association-id', 'Values': ['']}]
    )
    unused_eips = response.get('Addresses', [])
    monthly_waste = len(unused_eips) * 3.60

    return {
        "check_name": "unused_elastic_ips",
        "finding": f"{len(unused_eips)} unassociated Elastic IPs found",
        "monthly_waste_usd": monthly_waste,
        "resources": [eip['PublicIp'] for eip in unused_eips],
        "recommendation": "Release unassociated Elastic IPs to save "
                         f"${monthly_waste:.2f}/month",
        "severity": "HIGH" if monthly_waste > 10 else "LOW"
    }
```

```python
# infrastructure/terraform/modules/finops/lambda/handler.py
# Register the new analyzer:
from analyzers.unused_eips import analyze_unused_eips

ANALYZERS = [
    analyze_cost_by_service,
    analyze_compute_optimizer,
    analyze_tag_compliance,
    analyze_unused_eips,    # ← add here
]
```

After adding the analyzer, redeploy the Lambda:

```bash
cd infrastructure/terraform
terraform apply -var-file="environments/prod.tfvars" -target=module.finops
```
