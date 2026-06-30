# Plan My Journey — FinOps Operations Guide

## Overview

The FinOps system provides **automated cost monitoring, anomaly detection, and AI-powered optimization recommendations** for the Plan My Journey infrastructure. It runs hourly via AWS EventBridge and produces structured cost reports stored in DynamoDB.

Key capabilities:

- **Daily cost analysis**: MTD spend, daily trend, per-service breakdown
- **Anomaly detection**: Alerts when costs spike > 15% vs 7-day rolling average
- **AI recommendations**: Uses Bedrock Nova Pro to analyze cost patterns and suggest optimizations
- **Rightsizing**: Integrates with AWS Compute Optimizer for EC2/RDS instance recommendations
- **Tag compliance**: Scans all resources for required cost allocation tags
- **Real-time alerts**: EventBridge rules fire SNS notifications on unexpected resource creation

---

## Architecture

```
EventBridge Schedule (hourly — rate(1 hour))
          │
          ▼
  ┌─────────────────────────────────────────────────────┐
  │   Lambda: ai-travel-finops-cost-analyzer            │
  │   Runtime: Python 3.12  |  Memory: 512 MB           │
  │   Timeout: 5 minutes                                │
  │                                                     │
  │  Step 1: Cost Explorer API                          │
  │    ├── MTD total cost                               │
  │    ├── Daily cost for last 30 days                  │
  │    ├── Cost by AWS service                          │
  │    └── Cost by cost allocation tag (Project)        │
  │                                                     │
  │  Step 2: Compute Optimizer API                      │
  │    ├── EC2 instance recommendations                 │
  │    └── EBS volume recommendations                   │
  │                                                     │
  │  Step 3: CloudWatch Metrics                         │
  │    ├── EKS node CPU/memory (7-day avg)              │
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
```

---

## Cost Drivers and Baseline

### Major Infrastructure Costs

| Component | Notes |
|---|---|
| EKS control plane | Fixed monthly cost |
| EC2 worker nodes (Karpenter-managed) | Mix of Spot + On-Demand — Karpenter selects cheapest available |
| RDS Multi-AZ | ~2× cost of single-AZ; justified by RPO < 1 minute |
| NAT Gateways (× 2) | Per-AZ HA; data processing cost mitigated by VPC endpoints |
| VPC Endpoints (6 interface + 2 gateway) | Reduces NAT data processing charges significantly |
| Bedrock | Usage-based (tokens billed per invocation) |
| SQS, DynamoDB, CloudWatch | Low cost; scales with usage |

### Karpenter Spot Instance Savings

Karpenter actively selects Spot instances from a diverse set of instance families and sizes. This provides significant cost savings compared to fixed On-Demand node groups:

| Instance type | On-Demand price (us-east-1) | Spot price (typical) | Saving |
|---|---|---|---|
| t3.medium | ~$0.042/hr | ~$0.013–0.018/hr | 55–70% |
| m5.large | ~$0.096/hr | ~$0.029–0.040/hr | 58–70% |
| c5.large | ~$0.085/hr | ~$0.025–0.035/hr | 59–71% |

Because Karpenter considers multiple instance families simultaneously, it almost always finds cheap Spot capacity. When Spot is interrupted, Karpenter automatically falls back to On-Demand within seconds, so the application is unaffected.

Karpenter's consolidation policy (`WhenUnderUtilized`) additionally removes idle nodes — preventing the "minimum desired = 3" waste that Cluster Autoscaler can leave behind.

---

## Cost Alert Thresholds

| Alert Type | Threshold | Action |
|---|---|---|
| MTD cost limit | $500 | SNS → email |
| Daily anomaly | > 15% vs 7-day avg | SNS → email |
| Single service spike | > 50% day-over-day | SNS → email |
| Tag compliance | < 90% compliant | SNS → weekly digest |
| RDS storage | < 20% free space | CloudWatch alarm → SNS |

### Customizing Thresholds

Thresholds are configurable via Lambda environment variables. Update via Terraform:

```hcl
# PlanMyjourney-Terraform/modules/finops/lambda.tf
resource "aws_lambda_function" "cost_analyzer" {
  environment {
    variables = {
      COST_ALERT_THRESHOLD     = "750"
      ANOMALY_PCT_THRESHOLD    = "20"
      SERVICE_SPIKE_THRESHOLD  = "60"
      TAG_COMPLIANCE_MIN_PCT   = "95"
    }
  }
}
```

---

## Email Notifications

### Notification Schedule

| Notification | Trigger | Recipients |
|---|---|---|
| Cost anomaly alert | MTD exceeds threshold OR daily anomaly detected | SNS → configured alert email |
| Daily cost report | Every Lambda run | SES |
| Weekly digest | Monday 0:00 UTC | SES |
| Real-time alert | EC2/RDS/IAM change via EventBridge | SNS |
| Rightsizing PR | Compute Optimizer high-confidence recommendation | GitHub PR |

### Sample Email: Cost Anomaly Alert

```
Subject: [PlanMyJourney] Cost Anomaly Detected — $312 MTD (up 23% vs avg)

Plan My Journey — Daily Cost Report
Date: 2026-06-18 | Region: us-east-1

ANOMALY DETECTED
  Current MTD:   $312.40
  7-day avg MTD: $254.20
  Variance:      +$58.20 (+22.9%)

Top cost drivers today:
  1. NAT Gateway:   $8.40  (+41% vs avg) ← investigate data transfer
  2. RDS:           $5.90  (+12% vs avg)
  3. EKS nodes:     $4.80  (normal — Karpenter Spot instances)
  4. Bedrock:       $3.20  (normal)

AI Recommendation (powered by Bedrock):
  "The NAT Gateway cost spike suggests unusual outbound data transfer.
   Review CloudWatch VPC flow logs for the source pod or service generating
   unexpected traffic. Confirm VPC endpoints are active for S3, ECR, and
   Secrets Manager to avoid NAT charges for those AWS API calls."
```

### Sample Email: Weekly Digest

```
Subject: [PlanMyJourney] Weekly Cost Digest — Week of Jun 12–18, 2026

7-Day Summary:
  Total spend:        $79.20
  Budget:             $80.00  (99% utilized this week)
  Month-to-date:      $231.40
  Projected monthly:  $318.00

Top Services:
  EKS (control plane + nodes): $24.80  (31%) — Spot savings ~$18/week
  RDS Multi-AZ:                $14.70  (19%)
  NAT Gateways:                $17.20  (22%)
  VPC Endpoints:               $11.20  (14%)
  Other:                       $11.30  (14%)

Karpenter Spot Savings (estimated this week): ~$28.00

Optimization Opportunities:
  • RDS: Consider Graviton db.t4g.micro (-15% cost, similar performance)
  • Reserved Instance: t3.medium 1-year no-upfront saves ~$14/month

No anomalies detected this week.
```

---

## Reviewing Cost Recommendations

### Step 1: Read the DynamoDB Report

```bash
aws dynamodb scan \
  --table-name ai-travel-finops-reports \
  --filter-expression "#ts > :cutoff" \
  --expression-attribute-names '{"#ts": "timestamp"}' \
  --expression-attribute-values '{":cutoff": {"S": "2026-06-17"}}' \
  --region us-east-1 \
  --output json | jq '.Items[] | {timestamp: .timestamp.S, summary: .ai_summary.S}'
```

### Step 2: Read Rightsizing Recommendations

```bash
# EC2 recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --region us-east-1 \
  --query "instanceRecommendations[].{
    Instance: instanceArn,
    CurrentType: currentInstanceType,
    RecommendedType: recommendationOptions[0].instanceType,
    EstimatedSavings: recommendationOptions[0].estimatedMonthlySavings.value
  }" \
  --output table

# RDS recommendations
aws compute-optimizer get-rds-database-recommendations \
  --region us-east-1 \
  --output table
```

### Step 3: Review CloudWatch Dashboards

1. Open CloudWatch in `us-east-1`
2. Navigate to **Dashboards → ai-travel-finops**
3. Review: cost trend (30-day), service breakdown, EKS node utilization, RDS metrics, Karpenter node activity

### Step 4: Run FinOps Lambda Manually

```bash
aws lambda invoke \
  --function-name ai-travel-finops-cost-analyzer \
  --region us-east-1 \
  --log-type Tail \
  --output json \
  response.json

cat response.json | jq .

aws logs tail /aws/lambda/ai-travel-finops-prod --since 10m --region us-east-1
```

---

## Acting on Rightsizing Recommendations

### Manual Rightsizing: RDS Instance Type

```bash
# 1. Update Terraform variable
# PlanMyjourney-Terraform/environments/prod.tfvars
rds_instance_class = "db.t3.small"

# 2. Plan the change
terraform plan -var-file="environments/prod.tfvars" -target=module.rds

# 3. Apply during maintenance window (~60s Multi-AZ failover)
terraform apply -var-file="environments/prod.tfvars" -target=module.rds

# 4. Verify
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region us-east-1 \
  --query "DBInstances[0].{Class: DBInstanceClass, Status: DBInstanceStatus}"
```

### Manual Rightsizing: EKS Node Type

With Karpenter, the "node type" is determined by the NodePool `requirements`. To change the allowed instance families or sizes, update the NodePool manifest in `PlanMyJourney-Gitops/platform/karpenter/nodepool.yaml` and push — ArgoCD applies the change.

To explicitly request a different instance size for a specific workload, add a `nodeSelector` or `nodeAffinity` to the pod spec:

```yaml
nodeSelector:
  karpenter.k8s.aws/instance-size: large   # Force Karpenter to use a large instance
```

---

## Cost Optimization Playbook

### 1. Karpenter Node Rightsizing

Karpenter automatically selects the best-fitting instance, but you can guide it via NodePool requirements:

```bash
# Check what instance types Karpenter is currently using
kubectl get nodes -l karpenter.sh/nodepool=default \
  -o custom-columns=NAME:.metadata.name,TYPE:.metadata.labels.'karpenter\.k8s\.aws/instance-type',CAPACITY:.metadata.labels.'karpenter\.sh/capacity-type'

# If all nodes are On-Demand (e.g. Spot interrupted or capacity limited)
# Karpenter will retry Spot on next consolidation cycle — no action needed
```

### 2. NAT Gateway Cost Reduction

NAT Gateway data processing is often the largest variable cost driver. Strategies:

```bash
# Check NAT Gateway data usage per-AZ
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=<nat-gw-id> \
  --start-time 2026-06-11T00:00:00Z \
  --end-time 2026-06-18T00:00:00Z \
  --period 86400 \
  --statistics Sum \
  --region us-east-1

# Reduction strategies:
# 1. Verify all VPC endpoints are active (S3, ECR, Secrets Manager, DynamoDB, SSM)
# 2. Identify pods making excessive external API calls
#    kubectl exec -it <pod> -n prod -- curl -v https://external-api.com
```

### 3. Reserved Instance Purchase

For stable baseline capacity, Reserved Instances reduce On-Demand costs by 30–40%:

```bash
aws ce get-reservation-purchase-recommendation \
  --service "Amazon Elastic Compute Cloud - Compute" \
  --region us-east-1 \
  --output table
```

### 4. ECR Lifecycle Policy

```bash
# Keep only the last 20 tagged images per repository
aws ecr put-lifecycle-policy \
  --repository-name planmyjourney/user-service \
  --region us-east-1 \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "description": "Keep last 20 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["sha-"],
        "countType": "imageCountMoreThan",
        "countNumber": 20
      },
      "action": {"type": "expire"}
    }]
  }'
```

### 5. Unused Resource Cleanup

```bash
# Find unattached EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --region us-east-1 \
  --query "Volumes[].{VolumeId: VolumeId, Size: Size, Created: CreateTime}" \
  --output table

# Find old ECR images (untagged)
aws ecr list-images \
  --repository-name planmyjourney/user-service \
  --region us-east-1 \
  --filter tagStatus=UNTAGGED \
  --query "imageIds[].imageDigest"
```

---

## FinOps Lambda Environment Variables

| Variable | Description | Default |
|---|---|---|
| `NOTIFICATION_EMAIL` | Email address for cost alerts | (from Secrets Manager) |
| `COST_ALERT_THRESHOLD` | MTD cost alert threshold in USD | `500` |
| `ANOMALY_PCT_THRESHOLD` | % increase to trigger anomaly alert | `15` |
| `SERVICE_SPIKE_THRESHOLD` | % day-over-day increase per service | `50` |
| `TAG_COMPLIANCE_MIN_PCT` | Minimum % of resources with required tags | `90` |
| `AWS_REGION` | AWS region for all API calls | `us-east-1` |
| `DYNAMODB_TABLE` | DynamoDB table for report storage | `ai-travel-finops-reports` |
| `SNS_TOPIC_ARN` | SNS topic for anomaly alerts | (Terraform output) |
| `BEDROCK_MODEL_ID` | Bedrock model for AI analysis | `amazon.nova-pro-v1:0` |
| `REPORT_RETENTION_DAYS` | DynamoDB TTL for reports | `90` |
| `ENABLE_RIGHTSIZING_PRS` | Create GitHub PRs for rightsizing | `false` |
| `LOG_LEVEL` | Lambda log verbosity | `INFO` |

---

## Adding Custom Cost Checks

The FinOps Lambda is extensible. Add a new analyzer by creating a Python function and registering it in the analyzer chain.

### Example: Unused Elastic IP Checker

```python
# modules/finops/lambda/analyzers/unused_eips.py
import boto3
from typing import Dict, Any

def analyze_unused_eips(session: boto3.Session) -> Dict[str, Any]:
    """
    Check for Elastic IPs allocated but not associated with any instance.
    Each unassociated EIP costs ~$3.60/month.
    """
    ec2 = session.client('ec2', region_name='us-east-1')
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
        "recommendation": f"Release unassociated Elastic IPs to save ${monthly_waste:.2f}/month",
        "severity": "HIGH" if monthly_waste > 10 else "LOW"
    }
```

Register the analyzer in `handler.py`:

```python
from analyzers.unused_eips import analyze_unused_eips

ANALYZERS = [
    analyze_cost_by_service,
    analyze_compute_optimizer,
    analyze_tag_compliance,
    analyze_unused_eips,    # ← add here
]
```

Redeploy after adding:

```bash
terraform apply -var-file="environments/prod.tfvars" -target=module.finops
```
