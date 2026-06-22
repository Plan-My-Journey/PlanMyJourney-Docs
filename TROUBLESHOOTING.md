# Troubleshooting Guide

This guide covers the most common issues encountered when operating the AI-Travel-Planner
platform on EKS. Start with the Quick Diagnostics section before diving into specific issues.

---

## Quick Diagnostics

Run these commands first to get a holistic view of the system state:

```bash
# Overall node health
kubectl get nodes
# Expected: All nodes STATUS=Ready

# All unhealthy pods (anything not Running or Completed)
kubectl get pods -A | grep -v -E "Running|Completed"

# Application pods specifically
kubectl get pods -n ai-travel
kubectl get pods -n argocd
kubectl get pods -n kube-system

# Recent events (warnings and errors)
kubectl get events -n ai-travel --sort-by='.lastTimestamp' | tail -30

# ArgoCD application health
kubectl get applications -n argocd
# Or via CLI:
argocd app list

# Ingress status (is there an ALB address?)
kubectl get ingress -n ai-travel

# HPA scaling status
kubectl get hpa -n ai-travel

# Recent logs for a crashing pod
kubectl logs <pod-name> -n ai-travel --tail=100 --previous
```

---

## Issue 1: Pod CrashLoopBackOff

**Symptom:** Pod status shows `CrashLoopBackOff` — it keeps starting, crashing, and restarting.

**Diagnosis:**

```bash
# Step 1: Identify which pod is crashing
kubectl get pods -n ai-travel | grep -v Running

# Step 2: Check current logs (may be empty if container didn't start)
kubectl logs <pod-name> -n ai-travel

# Step 3: Check logs from the PREVIOUS run (before last crash)
kubectl logs <pod-name> -n ai-travel --previous

# Step 4: Describe pod for events and state
kubectl describe pod <pod-name> -n ai-travel
# Look for: Last State, Exit Code, Events section

# Step 5: Check if the container can start at all
kubectl run debug --image=<same-image> --rm -it --restart=Never -n ai-travel -- /bin/bash
```

**Common Causes and Fixes:**

| Exit Code | Likely Cause | Fix |
|-----------|-------------|-----|
| 1 | Application error at startup (Python traceback) | Check logs — usually missing env var or failed DB connection |
| 137 | OOMKilled (out of memory) | Increase memory limits in Helm values |
| 139 | Segmentation fault | Usually a bad native library — rebuild image |
| 2 | Shell/entrypoint script error | Check Dockerfile CMD/ENTRYPOINT |

**Missing secret causes CrashLoop:**

```bash
# Check if secrets CSI driver mounted the secret correctly
kubectl describe pod <pod-name> -n ai-travel | grep -A 5 "Mounts:"

# Verify secret exists in Secrets Manager
aws secretsmanager describe-secret \
  --secret-id "ai-travel/database-url" \
  --region ap-south-1

# Check SecretProviderClass is correct
kubectl get secretproviderclass -n ai-travel
kubectl describe secretproviderclass <name> -n ai-travel
```

---

## Issue 2: ImagePullBackOff / ErrImagePull

**Symptom:** Pod status shows `ImagePullBackOff` or `ErrImagePull` — Kubernetes cannot pull the container image.

**Diagnosis:**

```bash
# Check the error message
kubectl describe pod <pod-name> -n ai-travel | grep -A 10 "Events:"
# Look for: "Failed to pull image" or "unauthorized: authentication required"
```

**Fix A: ECR image does not exist**

```bash
# Verify the image tag exists in ECR
IMAGE_TAG=$(kubectl get pod <pod-name> -n ai-travel \
  -o jsonpath='{.spec.containers[0].image}' | cut -d: -f2)
echo "Looking for tag: $IMAGE_TAG"

aws ecr describe-images \
  --repository-name ai-travel/user-service \
  --region ap-south-1 \
  --image-ids imageTag=$IMAGE_TAG

# If not found: trigger a new CI build OR manually push:
docker pull user-service:latest
docker tag user-service:latest \
  ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/ai-travel/user-service:$IMAGE_TAG
docker push ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/ai-travel/user-service:$IMAGE_TAG
```

**Fix B: IRSA permissions missing for ECR pull**

```bash
# Verify the node IAM role has ECR pull permissions
# EKS nodes pull images using the EC2 instance profile, NOT IRSA
aws iam list-attached-role-policies \
  --role-name ai-travel-eks-node-role \
  --query "AttachedPolicies[].PolicyArn"
# Must include: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# If missing: attach the policy
aws iam attach-role-policy \
  --role-name ai-travel-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

**Fix C: Wrong AWS region in image URL**

```bash
# Image URL must use the same region as the cluster
kubectl get pod <pod-name> -n ai-travel \
  -o jsonpath='{.spec.containers[0].image}'
# Correct: ACCOUNT.dkr.ecr.ap-south-1.amazonaws.com/...
# Wrong:   ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/...  ← wrong region

# Fix: update image.repository in Helm values and sync ArgoCD
```

---

## Issue 3: Database Connection Errors

**Symptom:** Service pods crash with `Connection refused`, `FATAL: password authentication failed`,
or `could not connect to server` errors.

**Diagnosis:**

```bash
# Check RDS instance status
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region ap-south-1 \
  --query "DBInstances[0].{Status: DBInstanceStatus, Endpoint: Endpoint.Address}"

# Test connectivity from within the cluster
kubectl run pg-test \
  --image=postgres:15 \
  --rm -it \
  --restart=Never \
  -n ai-travel \
  -- psql -h <rds-endpoint> -U postgres -d user_db -c "SELECT 1"

# Check if DATABASE_URL secret is correctly set
aws secretsmanager get-secret-value \
  --secret-id "ai-travel/database-url" \
  --region ap-south-1 \
  --query "SecretString" \
  --output text
```

**Fix A: RDS Security Group not allowing EKS pods**

```bash
# Get EKS node security group ID
EKS_NODE_SG=$(aws eks describe-cluster \
  --name ai-travel-prod \
  --region ap-south-1 \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" \
  --output text)

# Get RDS security group ID
RDS_SG=$(aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region ap-south-1 \
  --query "DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId" \
  --output text)

# Verify rule exists: EKS nodes → RDS port 5432
aws ec2 describe-security-groups \
  --group-ids $RDS_SG \
  --query "SecurityGroups[0].IpPermissions[?FromPort==\`5432\`]"

# If missing: add the rule
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp \
  --port 5432 \
  --source-group $EKS_NODE_SG \
  --region ap-south-1
```

**Fix B: Alembic migration failure on pod startup**

```bash
# Check migration logs
kubectl logs <pod-name> -n ai-travel | grep -i alembic

# Run migration manually from within pod
kubectl exec -it <pod-name> -n ai-travel -- \
  alembic upgrade head

# Reset migrations (CAUTION: data loss in development only)
kubectl exec -it <pod-name> -n ai-travel -- \
  alembic downgrade base
kubectl exec -it <pod-name> -n ai-travel -- \
  alembic upgrade head
```

**Fix C: Too many connections to RDS**

```bash
# Check current connection count
kubectl exec -it <pod-name> -n ai-travel -- \
  psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity;"

# If near max (87 for db.t3.micro): scale down replicas temporarily
kubectl scale deployment user-service --replicas=1 -n ai-travel

# Long-term: add PgBouncer or reduce pool_size in SQLAlchemy config
```

---

## Issue 4: Bedrock Permission Denied (ai-service)

**Symptom:** `ai-service` logs show `AccessDeniedException` when calling Bedrock, or
`ai-service` returns static fallback responses instead of AI-generated content.

**Diagnosis:**

```bash
# Check ai-service logs for Bedrock errors
kubectl logs -l app=ai-service -n ai-travel --tail=50 | grep -i bedrock

# Verify IRSA is attached to the pod
kubectl get pod <ai-service-pod> -n ai-travel \
  -o jsonpath='{.spec.serviceAccountName}'
# Should be: ai-service-sa

kubectl get serviceaccount ai-service-sa -n ai-travel -o yaml
# Should have annotation: eks.amazonaws.com/role-arn: arn:aws:iam::...
```

**Fix A: Bedrock model access not enabled**

1. Go to [AWS Console → Bedrock → Model access](https://console.aws.amazon.com/bedrock/home?region=ap-south-1#/modelaccess) in `ap-south-1`
2. Find **Amazon Nova Pro** (`amazon.nova-pro-v1:0`)
3. Status must be **Access granted** (not "Available to request")
4. If not granted: click **Manage model access** → check Nova Pro → Submit

**Fix B: IRSA role missing bedrock:InvokeModel permission**

```bash
# Get the IRSA role ARN for ai-service
ROLE_ARN=$(kubectl get serviceaccount ai-service-sa -n ai-travel \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')
echo $ROLE_ARN

# Check the role's policies
aws iam list-role-policies --role-name ai-travel-ai-service
aws iam get-role-policy \
  --role-name ai-travel-ai-service \
  --policy-name BedrockInvokePolicy

# The policy must include:
# {
#   "Effect": "Allow",
#   "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
#   "Resource": "arn:aws:bedrock:ap-south-1::foundation-model/amazon.nova-pro-v1:0"
# }

# If wrong: fix in Terraform and apply
cd infrastructure/terraform
terraform apply -target=module.eks.aws_iam_role_policy.ai_service_bedrock
```

**Fix C: Verify IRSA token is being injected**

```bash
# Decode the IRSA token to verify it contains the correct role
kubectl exec -it <ai-service-pod> -n ai-travel -- \
  cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
  python3 -c "
import sys, base64, json
token = sys.stdin.read().split('.')[1]
# Add padding
token += '=' * (4 - len(token) % 4)
decoded = json.loads(base64.b64decode(token).decode())
print(json.dumps(decoded, indent=2))
"
# Look for: "eks.amazonaws.com/serviceaccount/namespace": "ai-travel"
# And: "sub": "system:serviceaccount:ai-travel:ai-service-sa"
```

**Fix D: Test Bedrock access from within the pod**

```bash
kubectl exec -it <ai-service-pod> -n ai-travel -- python3 -c "
import boto3, json
client = boto3.client('bedrock-runtime', region_name='ap-south-1')
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
# Check sync status
argocd app get frontend
argocd app get frontend --show-operation

# Get the detailed sync error
argocd app get frontend -o json | jq '.status.operationState.syncResult'
```

**Fix A: OutOfSync due to Helm defaults**

```bash
# See what ArgoCD thinks is different
argocd app diff frontend

# Common cause: Helm adds default values that aren't in your values.yaml
# Fix: add ignoreDifferences to the Application spec:
# spec:
#   ignoreDifferences:
#   - group: apps
#     kind: Deployment
#     jsonPointers:
#     - /spec/replicas    # Ignore if HPA manages replicas
```

**Fix B: Helm rendering error (check template syntax)**

```bash
# Reproduce locally
helm template frontend infrastructure/helm-charts/frontend/ \
  --values infrastructure/helm-charts/frontend/values.yaml \
  --debug 2>&1 | head -50

# Fix the template error and push to git
```

**Fix C: `Namespace "ai-travel" not found`**

```bash
# Create the namespace manually (or add CreateNamespace sync option)
kubectl create namespace ai-travel

# Add to ArgoCD app spec:
# syncOptions:
#   - CreateNamespace=true
```

---

## Issue 6: ALB Not Routing Traffic

**Symptom:** External URL returns connection refused or times out; ALB shows no healthy targets.

**Diagnosis:**

```bash
# Check if ingress has an ALB address
kubectl get ingress -n ai-travel
# ADDRESS column should contain the ALB DNS name

# If ADDRESS is empty: ALB controller is not working
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller \
  --tail=50

# Check ingress annotations
kubectl describe ingress frontend -n ai-travel

# Check target group health in AWS
ALB_ARN=$(aws elbv2 describe-load-balancers \
  --region ap-south-1 \
  --query "LoadBalancers[?contains(LoadBalancerName,'ai-travel')].LoadBalancerArn" \
  --output text)
TG_ARN=$(aws elbv2 describe-target-groups \
  --load-balancer-arn $ALB_ARN \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

**Fix A: ALB controller IRSA role missing permissions**

```bash
# The ALB controller needs specific IAM permissions
# Verify the IRSA role is correct
kubectl get serviceaccount aws-load-balancer-controller \
  -n kube-system -o yaml | grep role-arn

# Re-apply Terraform to ensure the IRSA role is correct
cd infrastructure/terraform
terraform apply -target=module.eks.module.alb_controller_irsa
```

**Fix B: Missing required annotations on Ingress**

```yaml
# Required annotations for ALB ingress to work:
annotations:
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443},{"HTTP":80}]'
  alb.ingress.kubernetes.io/ssl-redirect: "443"
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:...
```

---

## Issue 7: HPA Not Scaling

**Symptom:** Pods are overloaded (CPU > 90%) but HPA is not adding replicas.

**Diagnosis:**

```bash
# Check HPA status
kubectl describe hpa user-service -n ai-travel
# Look for: "unable to get metrics" or "Metrics not available"

# Check if metrics-server is running
kubectl get pods -n kube-system | grep metrics-server

# Check current metrics
kubectl top pods -n ai-travel
```

**Fix A: Metrics server not running**

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify it starts
kubectl wait --for=condition=available deployment/metrics-server \
  -n kube-system --timeout=60s
```

**Fix B: Resource requests not set on pods**

HPA calculates utilization as `current_usage / requested`. If `requests` is not set, HPA cannot
calculate a percentage and will not scale.

```yaml
# Required in Helm values:
resources:
  requests:
    cpu: "100m"      # HPA uses this as the baseline
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

---

## Issue 8: RDS Storage Full

**Symptom:** Application logs show `FATAL: could not write to file` or `ERROR: could not extend file`.

**Diagnosis:**

```bash
# Check current storage usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=ai-travel-prod \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average \
  --region ap-south-1
```

**Immediate Fix (< 5 minutes):**

```bash
# 1. Take a manual snapshot FIRST (in case the resize fails)
aws rds create-db-snapshot \
  --db-instance-identifier ai-travel-prod \
  --db-snapshot-identifier ai-travel-emergency-$(date +%Y%m%d) \
  --region ap-south-1

# 2. Modify storage immediately (no downtime for gp3)
aws rds modify-db-instance \
  --db-instance-identifier ai-travel-prod \
  --allocated-storage 100 \
  --apply-immediately \
  --region ap-south-1

# 3. Verify modification is in progress
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --query "DBInstances[0].PendingModifiedValues"
```

**Long-Term Fix:**

```hcl
# Enable auto-scaling in Terraform:
# infrastructure/terraform/modules/rds/main.tf
resource "aws_db_instance" "main" {
  max_allocated_storage = 500   # Auto-scale up to 500 GB
  allocated_storage     = 50    # Start at 50 GB
}
```

---

## Issue 9: High Monthly AWS Costs

**Symptom:** AWS bill exceeds expected $319/month baseline.

**Diagnosis:**

```bash
# Check MTD costs by service
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --region ap-south-1 \
  --query "ResultsByTime[0].Groups[].{Service: Keys[0], Cost: Metrics.BlendedCost.Amount}" \
  --output table | sort -k4 -rn

# Run FinOps Lambda for AI-powered recommendations
aws lambda invoke \
  --function-name ai-travel-finops-cost-analyzer \
  --region ap-south-1 \
  response.json && cat response.json | jq .
```

**Common Cost Culprits:**

| Cost Driver | How to Identify | Fix |
|-------------|----------------|-----|
| NAT Gateway data transfer | CloudWatch `BytesOutToDestination` metric | Verify VPC endpoints are active; check for pods making bulk external calls |
| CloudWatch Logs storage | `aws logs describe-log-groups --query '*[].storedBytes'` | Set 30-day retention on all log groups |
| Unused EBS snapshots | `aws ec2 describe-snapshots --owner-ids self` | Delete old manual snapshots |
| Over-provisioned nodes | `kubectl top nodes` shows <30% CPU | Reduce `node_group_desired_size` to 2 and let autoscaler manage |

---

## Issue 10: EKS Node NotReady

**Symptom:** `kubectl get nodes` shows a node with `STATUS=NotReady`.

**Diagnosis:**

```bash
# Describe the node for conditions
kubectl describe node <node-name>
# Look for: Conditions section, "Ready: False" reason

# Check node events
kubectl get events --field-selector reason=NodeNotReady

# SSH into node via SSM (no bastion needed)
aws ssm start-session \
  --target <ec2-instance-id> \
  --region ap-south-1

# Once on node: check kubelet status
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

**Fix A: Kubelet not running**

```bash
# On the node via SSM:
sudo systemctl restart kubelet
sudo systemctl enable kubelet
```

**Fix B: Node group failed to join cluster (new node)**

```bash
# Verify the node IAM role has required policies
aws iam list-attached-role-policies \
  --role-name ai-travel-eks-node-role \
  --query "AttachedPolicies[].PolicyName"
# Must include: AmazonEKSWorkerNodePolicy, AmazonEC2ContainerRegistryReadOnly,
#               AmazonEKS_CNI_Policy

# Check aws-auth ConfigMap includes the node role
kubectl describe configmap aws-auth -n kube-system
```

**Fix C: Force node replacement**

```bash
# Cordon and drain the bad node
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Terminate the EC2 instance (ASG will replace it)
aws ec2 terminate-instances --instance-ids <instance-id> --region ap-south-1

# Watch for replacement
kubectl get nodes -w
```

---

## Useful Debugging Commands Reference

```bash
# Port-forward to service directly (bypass ingress)
kubectl port-forward svc/user-service 8080:8000 -n ai-travel
# Test: curl http://localhost:8080/health

# Execute shell in running pod
kubectl exec -it <pod-name> -n ai-travel -- /bin/bash

# Execute one-off command
kubectl exec -it <pod-name> -n ai-travel -- python3 -c "import sys; print(sys.version)"

# Copy file from pod
kubectl cp ai-travel/<pod-name>:/app/logs/app.log /tmp/app.log

# Check service endpoints (are pods registered?)
kubectl get endpoints -n ai-travel

# Check persistent volumes
kubectl get pv,pvc -n ai-travel

# Describe a node for capacity and conditions
kubectl describe node <node-name>

# Check resource quotas and limits
kubectl describe resourcequota -n ai-travel
kubectl describe limitrange -n ai-travel

# Watch pod restarts in real-time
watch kubectl get pods -n ai-travel

# Check RBAC permissions for a service account
kubectl auth can-i list pods \
  --as=system:serviceaccount:ai-travel:user-service-sa \
  -n ai-travel

# Decode IRSA token (verify role binding)
kubectl exec -it <pod-name> -n ai-travel -- \
  cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
  python3 -c "
import sys, base64, json
token = sys.stdin.read().split('.')[1]
token += '=' * (4 - len(token) % 4)
print(json.dumps(json.loads(base64.b64decode(token).decode()), indent=2))
"
```

---

## Log Locations

| Log Type | Location | How to Access |
|----------|----------|---------------|
| Application logs | CloudWatch: `/aws/eks/ai-travel-prod/application` | `aws logs tail /aws/eks/ai-travel-prod/application --follow` |
| EKS control plane | CloudWatch: `/aws/eks/ai-travel-prod/cluster` | AWS Console → CloudWatch → Log groups |
| ALB access logs | S3: `ai-travel-alb-logs-{account}-ap-south-1` | `aws s3 sync s3://ai-travel-alb-logs-ACCOUNT-ap-south-1 /tmp/alb-logs` |
| CloudTrail | S3: `ai-travel-cloudtrail-{account}-ap-south-1` | AWS Console → CloudTrail → Event history |
| FinOps Lambda | CloudWatch: `/aws/lambda/ai-travel-finops-prod` | `aws logs tail /aws/lambda/ai-travel-finops-prod --follow` |
| VPC Flow Logs | CloudWatch: `/aws/vpc/flowlogs/ai-travel-prod` | Useful for debugging network connectivity issues |
