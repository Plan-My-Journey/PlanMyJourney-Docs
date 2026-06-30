# Plan My Journey — Troubleshooting Guide

This guide covers the most common issues encountered when operating the Plan My Journey platform on EKS. Start with Quick Diagnostics before diving into specific issues.

---

## Quick Diagnostics

Run these commands first to get a holistic view of the system state:

```bash
# Overall node health
kubectl get nodes

# All unhealthy pods (anything not Running or Completed)
kubectl get pods -A | grep -v -E "Running|Completed"

# Application pods specifically
kubectl get pods -n prod
kubectl get pods -n argocd
kubectl get pods -n kube-system
kubectl get pods -n karpenter
kubectl get pods -n keda

# Recent events (warnings and errors)
kubectl get events -n prod --sort-by='.lastTimestamp' | tail -30

# ArgoCD application health
kubectl get applications -n argocd

# Gateway status (is traffic routing?)
kubectl get gateway -n kgateway-system
kubectl get httproutes -n prod

# HPA and KEDA scaling status
kubectl get hpa -n prod
kubectl get scaledobjects -n prod

# Karpenter node provisioning
kubectl get nodepools
kubectl get nodeclaims

# Recent pod logs for a crashing pod
kubectl logs <pod-name> -n prod --tail=100 --previous
```

---

## Issue 1: Pod CrashLoopBackOff

**Symptom:** Pod status shows `CrashLoopBackOff` — it keeps starting, crashing, and restarting.

**Diagnosis:**

```bash
# Step 1: Identify which pod is crashing
kubectl get pods -n prod | grep -v Running

# Step 2: Check logs from the PREVIOUS run (before last crash)
kubectl logs <pod-name> -n prod --previous

# Step 3: Describe pod for events and state
kubectl describe pod <pod-name> -n prod
# Look for: Last State, Exit Code, Events section
```

**Common Causes and Fixes:**

| Exit Code | Likely Cause | Fix |
|---|---|---|
| 1 | Application error at startup (Python traceback) | Check logs — usually missing env var or failed DB connection |
| 137 | OOMKilled | Increase memory limits in `values-prod.yaml` |
| 139 | Segmentation fault | Rebuild image |
| 2 | Shell/entrypoint script error | Check Dockerfile `CMD`/`ENTRYPOINT` |

**Missing secret from Secrets Manager:**

```bash
# Verify the secret exists
aws secretsmanager describe-secret \
  --secret-id "ai-travel-prod/user-service" \
  --region us-east-1

# Check the pod's environment variables (secrets are injected as env vars via IRSA)
kubectl exec -it <pod-name> -n prod -- env | grep -E "DATABASE|SECRET|COGNITO"
```

---

## Issue 2: ImagePullBackOff / ErrImagePull

**Symptom:** Pod status shows `ImagePullBackOff` — Kubernetes cannot pull the container image.

**Diagnosis:**

```bash
kubectl describe pod <pod-name> -n prod | grep -A 10 "Events:"
# Look for: "Failed to pull image" or "unauthorized: authentication required"
```

**Fix A: ECR image does not exist**

```bash
IMAGE_TAG=$(kubectl get pod <pod-name> -n prod \
  -o jsonpath='{.spec.containers[0].image}' | cut -d: -f2)

aws ecr describe-images \
  --repository-name planmyjourney/user-service \
  --region us-east-1 \
  --image-ids imageTag=$IMAGE_TAG
```

If not found, trigger a new CI build or update `image.tag` in `values-prod.yaml` to a valid SHA.

**Fix B: Node IAM role missing ECR pull permissions**

```bash
# EKS nodes pull images using the EC2 instance profile
aws iam list-attached-role-policies \
  --role-name ai-travel-prod-eks-node-role \
  --query "AttachedPolicies[].PolicyArn"
# Must include: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# If missing:
aws iam attach-role-policy \
  --role-name ai-travel-prod-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

**Fix C: Wrong AWS region in image URL**

```bash
kubectl get pod <pod-name> -n prod -o jsonpath='{.spec.containers[0].image}'
# Correct: <account>.dkr.ecr.us-east-1.amazonaws.com/planmyjourney/...
# Wrong:   <account>.dkr.ecr.ap-south-1.amazonaws.com/...
```

Fix by updating `image.repository` in `values-prod.yaml` and pushing to the GitOps repo.

---

## Issue 3: Database Connection Errors

**Symptom:** Service pods crash with `Connection refused`, `FATAL: password authentication failed`, or `could not connect to server`.

**Diagnosis:**

```bash
# Check RDS instance status
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region us-east-1 \
  --query "DBInstances[0].{Status: DBInstanceStatus, Endpoint: Endpoint.Address}"

# Test connectivity from within the cluster
kubectl run pg-test \
  --image=postgres \
  --rm -it \
  --restart=Never \
  -n prod \
  -- psql -h <rds-endpoint> -U postgres -d user_db -c "SELECT 1"
```

**Fix A: RDS Security Group not allowing EKS traffic**

```bash
EKS_NODE_SG=$(aws eks describe-cluster \
  --name ai-travel-prod \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" \
  --output text)

RDS_SG=$(aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region us-east-1 \
  --query "DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId" \
  --output text)

# Verify port 5432 rule exists
aws ec2 describe-security-groups \
  --group-ids $RDS_SG \
  --query "SecurityGroups[0].IpPermissions[?FromPort==\`5432\`]"

# If missing:
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp --port 5432 \
  --source-group $EKS_NODE_SG \
  --region us-east-1
```

**Fix B: Alembic migration failure on pod startup**

```bash
kubectl logs <pod-name> -n prod | grep -i alembic

# Run migration manually inside the pod
kubectl exec -it <pod-name> -n prod -- alembic upgrade head
```

**Fix C: Too many connections to RDS**

```bash
# Check current connection count
kubectl exec -it <pod-name> -n prod -- \
  psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity;"

# Temporarily reduce replicas
kubectl scale deployment user-service --replicas=1 -n prod

# Long-term: reduce pool_size in SQLAlchemy config or upgrade RDS instance class
```

---

## Issue 4: Bedrock Permission Denied (ai-service)

**Symptom:** `ai-service` logs show `AccessDeniedException` or return static fallback responses instead of AI-generated itineraries.

**Diagnosis:**

```bash
# Check ai-service logs for Bedrock errors
kubectl logs -l app.kubernetes.io/name=ai-service -n prod --tail=50 | grep -i -E "bedrock|access|denied|error"

# Verify IRSA is attached to the pod
kubectl get serviceaccount ai-service-sa -n prod -o yaml
# Should have annotation: eks.amazonaws.com/role-arn: arn:aws:iam::...

# Check which environment variables the pod sees
kubectl exec -it <ai-service-pod> -n prod -- env | grep -E "BEDROCK|AWS"
```

**Fix A: Bedrock model access not enabled**

1. Go to AWS Console → Bedrock → Model access in `us-east-1`
2. Find **Amazon Nova Pro** (`amazon.nova-pro-v1:0`)
3. Status must be **Access granted** — if not, click **Manage model access** → enable Nova Pro → Submit

**Fix B: IRSA role missing bedrock:InvokeModel permission**

```bash
ROLE_ARN=$(kubectl get serviceaccount ai-service-sa -n prod \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')
echo $ROLE_ARN

aws iam list-role-policies --role-name ai-travel-prod-ai-service
# Must include a policy with "bedrock:InvokeModel" on the Nova Pro model ARN

# Fix in Terraform:
terraform apply -target=module.eks.aws_iam_role_policy.ai_service_bedrock
```

**Fix C: Test Bedrock access from within the pod**

```bash
kubectl exec -it <ai-service-pod> -n prod -- python3 -c "
import boto3, json
client = boto3.client('bedrock-runtime', region_name='us-east-1')
response = client.converse(
    modelId='amazon.nova-pro-v1:0',
    messages=[{'role': 'user', 'content': [{'text': 'Hello, are you working?'}]}]
)
print(response['output']['message']['content'][0]['text'])
"
```

---

## Issue 5: ArgoCD Sync Failures

**Symptom:** ArgoCD shows an application as `OutOfSync` or sync fails with an error.

**Diagnosis:**

```bash
argocd app get ai-service-prod --show-operation
argocd app get ai-service-prod -o json | jq '.status.operationState.syncResult'
argocd app diff ai-service-prod
```

**Fix A: OutOfSync due to Helm defaults**

```bash
# See what ArgoCD thinks is different
argocd app diff ai-service-prod

# Common fix: add ignoreDifferences to the Application spec for HPA-managed replicas
# spec:
#   ignoreDifferences:
#   - group: apps
#     kind: Deployment
#     jsonPointers:
#     - /spec/replicas
```

**Fix B: Helm rendering error**

```bash
helm template ai-service helm-charts/ai-service/ \
  --values helm-charts/ai-service/values-prod.yaml \
  --debug 2>&1 | head -50
```

Fix the template error and push to the GitOps repo.

**Fix C: Namespace not found**

```bash
kubectl create namespace prod

# Or add to the Application spec:
# syncOptions:
#   - CreateNamespace=true
```

---

## Issue 6: KGateway — Traffic Not Routing

**Symptom:** External URL returns connection refused or 502; requests are not reaching backend pods.

**Diagnosis:**

```bash
# Check Gateway status
kubectl get gateway -n kgateway-system
# PROGRAMMED column should show: True

# Check HTTPRoute status
kubectl get httproutes -n prod
# CONDITIONS column: ResolvedRefs=True for all routes

# Check if KGateway Envoy pods are running
kubectl get pods -n kgateway-system

# Check NLB address (should be a DNS name)
kubectl get gateway api-gateway -n kgateway-system \
  -o jsonpath='{.status.addresses[0].value}'

# Check KGateway controller logs
kubectl logs -n kgateway-system \
  -l app.kubernetes.io/name=kgateway --tail=50
```

**Fix A: Gateway not programmed (NLB not created)**

```bash
# Check if the gateway controller has the correct IRSA permissions
kubectl get serviceaccount -n kgateway-system -o yaml | grep role-arn

# Look for NLB creation errors in controller logs
kubectl logs -n kgateway-system \
  -l app.kubernetes.io/name=kgateway --tail=100 | grep -i -E "error|nlb|load balancer"

# Re-sync the KGateway ArgoCD application
argocd app sync kgateway-prod
```

**Fix B: HTTPRoute not resolving to backend**

```bash
# Check if the backend service exists
kubectl get service -n prod
# The HTTPRoute backendRef service name must match exactly

kubectl describe httproute <route-name> -n prod
# Events section shows resolution errors
```

**Fix C: TLS certificate issue (NLB listener)**

```bash
# Verify ACM certificate is issued and in us-east-1
aws acm list-certificates --region us-east-1 \
  --query "CertificateSummaryList[?DomainName=='invest-iq.online']"

# Check NLB listener has the correct certificate ARN
aws elbv2 describe-listeners \
  --load-balancer-arn <nlb-arn> \
  --region us-east-1 \
  --query "Listeners[?Port==\`443\`].Certificates"
```

**Fix D: Route 53 not pointing to NLB**

```bash
# Get the NLB DNS name
NLB_DNS=$(kubectl get gateway api-gateway -n kgateway-system \
  -o jsonpath='{.status.addresses[0].value}')
echo $NLB_DNS

# Compare with Route 53 record
aws route53 list-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --query "ResourceRecordSets[?Name=='invest-iq.online.']"
```

---

## Issue 7: KEDA — ai-worker Not Scaling

**Symptom:** SQS queue has messages but `ai-worker` pods are not starting, or KEDA is not scaling as expected.

**Diagnosis:**

```bash
# Check ScaledObject status
kubectl describe scaledobject ai-worker-scaledobject -n prod
# Look for: "SuccessfullySet" or error messages in Events

# Check KEDA operator logs
kubectl logs -n keda -l app.kubernetes.io/name=keda-operator --tail=100

# Check queue depth manually
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/235270183260/ai-travel-prod-ai-jobs \
  --attribute-names ApproximateNumberOfMessages \
  --region us-east-1

# Check TriggerAuthentication
kubectl describe triggerauthentication keda-trigger-auth-aws-irsa -n prod
```

**Fix A: KEDA IRSA role missing SQS permissions**

```bash
# Get the KEDA operator's IRSA role
kubectl get serviceaccount keda-operator -n keda -o yaml | grep role-arn

# Verify it has sqs:GetQueueAttributes
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<account>:role/ai-travel-prod-keda-operator \
  --action-names sqs:GetQueueAttributes \
  --resource-arns arn:aws:sqs:us-east-1:<account>:ai-travel-prod-ai-jobs

# Fix in Terraform:
terraform apply -target=module.keda_irsa
```

**Fix B: KEDA cannot authenticate to AWS (OIDC issue)**

```bash
# Verify the KEDA operator pod's service account has the IRSA annotation
kubectl get pod -n keda -l app.kubernetes.io/name=keda-operator \
  -o jsonpath='{.items[0].spec.serviceAccountName}'
# Should be: keda-operator

kubectl get serviceaccount keda-operator -n keda -o yaml
# Must have: eks.amazonaws.com/role-arn annotation
```

**Fix C: ScaledObject targeting wrong deployment**

```bash
# Verify the scaleTargetRef name matches the actual deployment name
kubectl get scaledobject ai-worker-scaledobject -n prod \
  -o jsonpath='{.spec.scaleTargetRef.name}'
# Must match:
kubectl get deployment -n prod | grep ai-worker
```

---

## Issue 8: Karpenter — Nodes Not Provisioning

**Symptom:** Pods remain in `Pending` state for more than 2 minutes; no new nodes appear.

**Diagnosis:**

```bash
# Check Karpenter logs for provisioning attempts
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=100 | grep -i -E "launch|error|nodeclaim"

# Check NodePool limits (has the CPU/memory ceiling been reached?)
kubectl describe nodepool default | grep -A 20 "Status:"

# Check the pending pod resource requests
kubectl describe pod <pending-pod> -n prod | grep -A 10 "Requests:"

# Check EC2 subnet tags (Karpenter discovers subnets via tags)
aws ec2 describe-subnets \
  --filters "Name=tag:karpenter.sh/discovery,Values=ai-travel-prod" \
  --query 'Subnets[].SubnetId' \
  --region us-east-1
```

**Fix A: NodePool limits reached**

```bash
# Check current usage vs limits
kubectl describe nodepool default | grep -A 5 "Resources"

# Increase limits in the NodePool manifest in PlanMyJourney-Gitops
# platform/karpenter/nodepool.yaml
# limits:
#   cpu: "96"         # increased from 48
#   memory: 192Gi

# Push to Git — ArgoCD applies it
git push origin main
```

**Fix B: No Spot capacity (instance interruption)**

Karpenter automatically falls back from Spot to On-Demand when Spot capacity is unavailable. If you see `InsufficientInstanceCapacity` in Karpenter logs for all Spot types, Karpenter will provision On-Demand until Spot capacity recovers. No action needed.

**Fix C: Subnet or security group discovery tags missing**

```bash
# Tags must exist on private subnets and EKS security groups:
aws ec2 describe-subnets \
  --filters "Name=tag:karpenter.sh/discovery,Values=ai-travel-prod" \
  --region us-east-1

# If missing, add tags via Terraform:
resource "aws_subnet" "private" {
  tags = {
    "karpenter.sh/discovery" = "ai-travel-prod"
  }
}
```

---

## Issue 9: HPA Not Scaling

**Symptom:** Pods are overloaded (CPU > 90%) but HPA is not adding replicas.

**Diagnosis:**

```bash
kubectl describe hpa user-service -n prod
# Look for: "unable to get metrics" or "Metrics not available"

kubectl get pods -n kube-system | grep metrics-server
kubectl top pods -n prod
```

**Fix A: metrics-server not running**

```bash
# Check ArgoCD metrics-server application
argocd app get metrics-server-prod

# Re-sync if needed
argocd app sync metrics-server-prod

# Verify it's healthy
kubectl rollout status deployment/metrics-server -n kube-system
```

**Fix B: Resource requests not set on pods**

HPA calculates utilization as `current_usage / requested`. If `resources.requests` is not set, HPA cannot calculate a ratio.

```yaml
# Required in values-prod.yaml:
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

---

## Issue 10: RDS Storage Full

**Symptom:** Application logs show `FATAL: could not write to file` or `ERROR: could not extend file`.

**Diagnosis:**

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=ai-travel-prod \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average \
  --region us-east-1
```

**Immediate Fix (no downtime for gp3):**

```bash
# 1. Take a manual snapshot first
aws rds create-db-snapshot \
  --db-instance-identifier ai-travel-prod \
  --db-snapshot-identifier ai-travel-emergency-$(date +%Y%m%d) \
  --region us-east-1

# 2. Increase storage
aws rds modify-db-instance \
  --db-instance-identifier ai-travel-prod \
  --allocated-storage 100 \
  --apply-immediately \
  --region us-east-1
```

**Long-Term Fix:**

```hcl
# PlanMyjourney-Terraform/modules/rds/main.tf
resource "aws_db_instance" "main" {
  max_allocated_storage = 500   # Auto-scale up to 500 GB
  allocated_storage     = 50
}
```

---

## Issue 11: RDS Multi-AZ Failover

**Symptom:** Brief application connection errors (< 2 minutes); RDS primary is in a different AZ than expected.

**Behaviour:**

RDS Multi-AZ failover is fully automatic. Route 53 updates the CNAME to point to the new primary, and the standby is promoted. Applications using SQLAlchemy with `pool_pre_ping=True` reconnect automatically after the failover.

**Verification:**

```bash
# Check which AZ the primary is in
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region us-east-1 \
  --query "DBInstances[0].{AZ: AvailabilityZone, Status: DBInstanceStatus, SecondaryAZ: SecondaryAvailabilityZone}"

# Check recent RDS events (failover appears here)
aws rds describe-events \
  --source-identifier ai-travel-prod \
  --source-type db-instance \
  --duration 60 \
  --region us-east-1
```

**If applications do not reconnect automatically:**

```bash
# Rolling restart of all services to force reconnection
for svc in frontend user-service travel-service ai-service ai-worker utility-service; do
  kubectl rollout restart deployment/$svc -n prod
done
```

---

## Issue 12: High Monthly AWS Costs

**Symptom:** AWS bill exceeds expected monthly baseline.

**Diagnosis:**

```bash
# Check MTD costs by service
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --region us-east-1 \
  --query "ResultsByTime[0].Groups[].{Service: Keys[0], Cost: Metrics.BlendedCost.Amount}" \
  --output table

# Run FinOps Lambda for AI-powered recommendations
aws lambda invoke \
  --function-name ai-travel-finops-cost-analyzer \
  --region us-east-1 \
  response.json && cat response.json | jq .
```

**Common Cost Culprits:**

| Cost Driver | How to Identify | Fix |
|---|---|---|
| NAT Gateway data transfer | CloudWatch `BytesOutToDestination` | Verify all VPC endpoints active; check for pods making bulk external calls |
| Karpenter using On-Demand instead of Spot | `kubectl get nodes -l karpenter.sh/capacity-type=on-demand` | Check Karpenter logs; Spot availability usually recovers automatically |
| CloudWatch Logs storage | `aws logs describe-log-groups` | Set 7-day retention for dev, 30-day for prod log groups |
| Over-provisioned RDS | Compute Optimizer recommendations | Downsize instance class after analyzing CPU metrics |
| Unused EBS volumes | `aws ec2 describe-volumes --filters Name=status,Values=available` | Delete unattached volumes |

---

## Useful Debugging Commands Reference

```bash
# Port-forward to a service directly (bypass KGateway)
kubectl port-forward svc/user-service 8080:8000 -n prod
# Test: curl http://localhost:8080/health

# Execute shell in running pod
kubectl exec -it <pod-name> -n prod -- /bin/bash

# Copy a file from pod to local
kubectl cp prod/<pod-name>:/app/logs/app.log /tmp/app.log

# Check service endpoints (are pods registered?)
kubectl get endpoints -n prod

# Describe a node for capacity and conditions
kubectl describe node <node-name>

# Watch pod restarts in real-time
watch kubectl get pods -n prod

# Check RBAC permissions for a service account
kubectl auth can-i list pods \
  --as=system:serviceaccount:prod:user-service-sa \
  -n prod

# Decode IRSA token to verify role binding
kubectl exec -it <pod-name> -n prod -- \
  cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
  python3 -c "
import sys, base64, json
token = sys.stdin.read().split('.')[1]
token += '=' * (4 - len(token) % 4)
print(json.dumps(json.loads(base64.b64decode(token).decode()), indent=2))
"
# Look for: "sub": "system:serviceaccount:prod:<service-sa>"

# SSM Session Manager — access EKS nodes without a bastion
aws ssm start-session --target <ec2-instance-id> --region us-east-1
```

---

## Log Locations

| Log Type | Location | How to Access |
|---|---|---|
| Application logs (kubectl) | Pod stdout/stderr | `kubectl logs <pod> -n prod --tail=100 -f` |
| EKS control plane | CloudWatch: `/aws/eks/ai-travel-prod/cluster` | AWS Console → CloudWatch → Log groups |
| Karpenter controller | Kubernetes pod logs | `kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter -f` |
| KEDA operator | Kubernetes pod logs | `kubectl logs -n keda -l app.kubernetes.io/name=keda-operator -f` |
| KGateway controller | Kubernetes pod logs | `kubectl logs -n kgateway-system -l app.kubernetes.io/name=kgateway -f` |
| CloudTrail (all AWS API calls) | S3 bucket + CloudTrail console | AWS Console → CloudTrail → Event history |
| FinOps Lambda | CloudWatch: `/aws/lambda/ai-travel-finops-prod` | `aws logs tail /aws/lambda/ai-travel-finops-prod --follow --region us-east-1` |
| VPC Flow Logs | CloudWatch: `/aws/vpc/flowlogs/ai-travel-prod` | Useful for debugging unexpected network traffic or NAT cost spikes |
