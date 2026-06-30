# Plan My Journey — Scaling Guide

## Overview

The Plan My Journey platform uses three complementary layers of scaling to handle variable traffic while maintaining cost efficiency:

| Layer | Technology | What Scales | Trigger |
|-------|-----------|-------------|---------|
| **Pod (event-driven)** | KEDA | `ai-worker` replicas | SQS queue depth |
| **Pod (CPU-based)** | Kubernetes HPA | All service replicas | CPU utilisation |
| **Node** | Karpenter | EC2 nodes | Unschedulable pods |

The general scaling philosophy:

1. **Scale pods first** — fastest, zero-downtime, cheapest
2. **Scale nodes on demand** — Karpenter provisions new nodes in < 60 seconds when pods can't be scheduled
3. **KEDA drives cost efficiency** — `ai-worker` scales to zero when no jobs are queued, eliminating idle worker cost

---

## Event-Driven Pod Autoscaling — KEDA

KEDA (Kubernetes Event Driven Autoscaler) extends the standard Kubernetes HPA with support for external event sources. For this platform, KEDA scales pods based on SQS queue depth.

### Why KEDA Over HPA for ai-worker

CPU is not a useful scaling signal for `ai-worker`. The worker spends most of its time waiting on Bedrock responses — CPU stays low even when dozens of jobs are queued. KEDA solves this by scaling based on the actual work to be done: messages in the SQS queue.

### ScaledObject — ai-worker

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ai-worker-scaledobject
  namespace: prod
spec:
  scaleTargetRef:
    name: ai-worker
  pollingInterval: 30          # Check queue every 30 seconds
  cooldownPeriod: 120          # Wait 120s idle before scaling down
  minReplicaCount: 0           # Scale to zero when queue is empty
  maxReplicaCount: 10
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/235270183260/ai-travel-prod-ai-jobs
      queueLength: "5"         # 1 pod per 5 queued messages
      awsRegion: us-east-1
    authenticationRef:
      name: keda-trigger-auth-aws-irsa
```

### ScaledObject — ai-service

`ai-service` also has a KEDA ScaledObject, maintaining a floor of 2 replicas for latency but scaling up when the queue grows deep:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ai-service-scaledobject
  namespace: prod
spec:
  scaleTargetRef:
    name: ai-service
  minReplicaCount: 2
  maxReplicaCount: 10
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/235270183260/ai-travel-prod-ai-jobs
      queueLength: "10"
      awsRegion: us-east-1
    authenticationRef:
      name: keda-trigger-auth-aws-irsa
```

### TriggerAuthentication (IRSA — no static credentials)

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-aws-irsa
  namespace: prod
spec:
  podIdentity:
    provider: aws-eks
```

KEDA uses the `keda-operator` service account's IRSA role to call `sqs:GetQueueAttributes`. No AWS access keys are stored anywhere.

### KEDA Scaling Timeline

```
[Message arrives in SQS queue]
        │
        ▼ KEDA polls queue every 30s
[KEDA calculates: messages / queueLength = desired pods]
        │
        ▼ (if ai-worker was at 0)
[KEDA creates HPA object targeting ai-worker]
[Kubernetes schedules pod]
        │
        ▼ (~30–60 seconds total)
[ai-worker pod running — pulls message — calls Bedrock]
        │
        ▼ (queue empty for 120s cooldown period)
[KEDA scales ai-worker back to 0]
```

### Monitoring KEDA

```bash
# View ScaledObjects and their status
kubectl get scaledobjects -n prod

# Detailed view — shows current replica count and active state
kubectl describe scaledobject ai-worker-scaledobject -n prod

# Check KEDA operator logs
kubectl logs -n keda -l app.kubernetes.io/name=keda-operator --tail=100

# View KEDA metrics
kubectl get hpa -n prod   # KEDA creates/manages an HPA for each ScaledObject

# Check queue depth manually
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/235270183260/ai-travel-prod-ai-jobs \
  --attribute-names ApproximateNumberOfMessages \
  --region us-east-1
```

---

## Horizontal Pod Autoscaling — HPA

HPA manages CPU-based scaling for all services except `ai-worker` (which uses KEDA).

### Current HPA Configuration

| Service | Min Replicas | Max Replicas | CPU Target | Notes |
|---------|-------------|-------------|-----------|-------|
| `frontend` | 2 | 10 | 60% | Serving static assets + SSR |
| `user-service` | 2 | 5 | 70% | DB-bound — auth and user profile |
| `travel-service` | 2 | 5 | 70% | DB-bound — trip CRUD |
| `utility-service` | 2 | 5 | 70% | External API calls — weather, hotels |
| `ai-service` | 2 | 10 | 70% | I/O-bound — Bedrock calls |

`ai-worker` is managed entirely by KEDA and has no independent HPA.

### HPA Scaling Timeline

```
T+0s    Pod CPU exceeds threshold (e.g., 70%)
T+15s   HPA queries metrics-server (15-second scrape interval)
T+15s   HPA calculation: current=85%, target=70%, ratio=1.21
T+15s   HPA decision: ceil(2 × 1.21) = 3 replicas
T+30s   New pod scheduled — Karpenter provisions a node if needed
T+60s   New pod passes readiness probe, starts receiving traffic

--- Load decreases ---
T+350s  CPU below threshold for 5+ minutes (scale-down stabilisation)
T+360s  Old pod gracefully terminated (SIGTERM → 30s grace period)
```

### HPA Manifest Reference

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60             # Remove at most 1 pod per minute
    scaleUp:
      stabilizationWindowSeconds: 0    # Scale up immediately
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60            # Add up to 2 pods per minute
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
```

### Monitoring HPA

```bash
# View HPA status and current metrics
kubectl get hpa -n prod

# Detailed status (shows scaling events)
kubectl describe hpa user-service -n prod

# Watch HPA in real-time
watch kubectl get hpa -n prod

# Current pod resource usage
kubectl top pods -n prod --sort-by=cpu
kubectl top pods -n prod --sort-by=memory

# Historical scaling events
kubectl get events -n prod \
  --field-selector reason=SuccessfulRescale \
  --sort-by='.lastTimestamp'
```

### Tuning HPA

| Service Type | Recommended CPU Target | Reason |
|-------------|----------------------|--------|
| CPU-intensive (compute) | 60–70% | Leave headroom for sudden spikes |
| I/O-intensive (DB calls) | 70–80% | CPU stays low even under load |
| Long-running requests (AI) | 70% | Requests queue up fast at high load |

If `kubectl get hpa` shows replicas constantly at the max with CPU still above target, increase `maxReplicas` in the service's `values-prod.yaml` and push to the GitOps repo. ArgoCD applies the change within 3 minutes.

---

## Node Autoscaling — Karpenter

Karpenter replaces the traditional Cluster Autoscaler. It provisions EC2 nodes in under 60 seconds and actively consolidates underutilised nodes to reduce cost.

### How Karpenter Works

```
[Pod in Pending state — cannot be scheduled on existing nodes]
        │
        ▼ Karpenter watches for unschedulable pods
Karpenter evaluates NodePool constraints and pod requirements
        │
        ▼ (selects cheapest Spot or On-Demand instance that satisfies requirements)
Karpenter calls EC2 CreateFleet directly (bypasses ASG API delay)
        │
        ▼ (< 60 seconds from Pending to Running)
New node joins cluster, pod is scheduled and starts
        │
        ▼ (when node utilisation drops below threshold for 30s)
Karpenter consolidates — reschedules pods to other nodes, terminates the idle node
```

### NodePool Configuration

The NodePool defines which instance families and sizes Karpenter may use, and sets the consolidation policy:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["t3", "t3a", "m5", "m5a", "c5"]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["medium", "large", "xlarge"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1
        kind: EC2NodeClass
        name: default
      expireAfter: 720h    # Rotate nodes every 30 days for AMI freshness
  limits:
    cpu: "48"
    memory: 96Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
```

### EC2NodeClass Configuration

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  role: ai-travel-prod-karpenter-node
  subnetSelectorTerms:
    - tags:
        kubernetes.io/role/internal-elb: "1"
        karpenter.sh/discovery: ai-travel-prod
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ai-travel-prod
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        encrypted: true
```

### Karpenter vs Cluster Autoscaler

| Feature | Karpenter | Cluster Autoscaler |
|---|---|---|
| Node provisioning speed | < 60 seconds | 3–5 minutes |
| Instance type selection | Any (selects cheapest that fits) | Fixed instance types in node group |
| Spot instance support | Native, automatic fallback to On-Demand | Separate node group required |
| Consolidation | Active (moves pods, terminates node) | Reactive only |
| AMI freshness | Node drift detection + expiry policy | Manual update |

### Pod Disruption Budgets

Karpenter respects PodDisruptionBudgets when consolidating nodes. All services have a PDB with `minAvailable: 1`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
  namespace: prod
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: user-service
```

This ensures Karpenter never terminates a node that would take down the last running pod of any service.

### Monitoring Karpenter

```bash
# View Karpenter-managed nodes
kubectl get nodes -l karpenter.sh/nodepool=default

# View NodePool status (limits, usage, conditions)
kubectl get nodepools
kubectl describe nodepool default

# View Karpenter controller logs
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=100 -f

# Key log lines to watch for:
# "launching nodeclaim" — a new node is being provisioned
# "disrupting node" — Karpenter is consolidating a node
# "registered nodeclaim" — node joined cluster successfully

# Check NodeClaims (Karpenter's abstraction for EC2 instances it manages)
kubectl get nodeclaims

# See why a pod is still Pending
kubectl describe pod <pod-name> -n prod | grep -A 5 "Events:"
```

### When Node Provisioning is Slow

If pods remain in `Pending` for more than 2 minutes, check:

```bash
# 1. Check if Karpenter is attempting to launch
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=50 | grep -i launch

# 2. Check NodePool limits (is max CPU/memory reached?)
kubectl describe nodepool default | grep -A 10 "Status:"

# 3. Check pod resource requests (Karpenter needs these to select an instance)
kubectl describe pod <pending-pod> -n prod | grep -A 5 "Requests:"

# 4. Check for spot capacity issues (Karpenter will fall back to On-Demand automatically)
# Look for: "InsufficientInstanceCapacity" in Karpenter logs
```

---

## Database Scaling (RDS)

### Current RDS Configuration

| Parameter | Value |
|-----------|-------|
| Instance class | db.t3.micro |
| Multi-AZ | Enabled (synchronous standby) |
| Storage | 50 GB gp3, auto-scales to 100 GB |
| IOPS | 3,000 baseline (gp3) |
| Throughput | 125 MB/s baseline (gp3) |

### Vertical Scaling (Upgrading Instance Class)

RDS vertical scaling with Multi-AZ causes a brief automated failover (~60 seconds):

```
db.t3.micro (2 vCPU, 1 GB)  →  db.t3.small (2 vCPU, 2 GB)  →  db.t3.medium (2 vCPU, 4 GB)
```

To perform the upgrade:

```bash
# 1. Update the Terraform variable
# PlanMyjourney-Terraform/environments/prod.tfvars
db_instance_class = "db.t3.small"

# 2. Plan and review
terraform plan -var-file="environments/prod.tfvars" -target=module.rds

# 3. Apply during a low-traffic window
terraform apply -var-file="environments/prod.tfvars" -target=module.rds

# 4. Monitor modification
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region us-east-1 \
  --query "DBInstances[0].{Status: DBInstanceStatus, Class: DBInstanceClass}"
```

SQLAlchemy's `pool_pre_ping=True` handles the brief connection reset during failover transparently.

### Connection Pool Considerations

As pods scale horizontally, connection pool settings matter:

| Instance | Max connections | Safe pods per service (pool_size=10) |
|---|---|---|
| db.t3.micro | 87 | 2 |
| db.t3.small | 193 | 6 |
| db.t3.medium | 415 | 13 |

If you need more pods than the current instance supports: reduce `pool_size` in SQLAlchemy config, or upgrade the instance class.

---

## Load Testing

### Basic Health Endpoint Stress Test (k6)

```javascript
// tests/load/health-check.js
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 10 },
    { duration: '60s', target: 50 },
    { duration: '60s', target: 50 },
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const response = http.get('https://api.invest-iq.online/health');
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  sleep(1);
}
```

### Observing Scaling During Load Test

```bash
# Watch pods scale (HPA + KEDA)
watch kubectl get hpa,scaledobjects,pods -n prod

# Watch node provisioning (Karpenter)
watch kubectl get nodes

# Watch resource utilisation
watch kubectl top pods -n prod
```

---

## Performance Benchmarks

Target latencies under normal load (< 50 concurrent users):

| Service | p50 | p95 | p99 | Notes |
|---------|-----|-----|-----|-------|
| `frontend` | < 50ms | < 200ms | < 500ms | Static serving |
| `user-service` | < 100ms | < 500ms | < 1,000ms | DB-bound |
| `travel-service` | < 100ms | < 400ms | < 800ms | DB-bound |
| `ai-service` | < 2,000ms | < 5,000ms | < 10,000ms | Bedrock latency |
| `utility-service` | < 200ms | < 1,000ms | < 2,000ms | External API latency |

---

## Traffic Spike Runbook

```
1. CHECK current state
   kubectl get hpa,scaledobjects -n prod       # Is autoscaling active?
   kubectl top pods -n prod                    # Which service is overloaded?
   kubectl get nodes                            # Are nodes available?

2. IDENTIFY the overloaded service
   Look for CPU > 80% in `kubectl top pods` output

3. MANUAL SCALE (if autoscaling is slow to react)
   kubectl scale deployment user-service --replicas=4 -n prod

4. CHECK if nodes are the bottleneck
   kubectl get pods -n prod | grep Pending
   # If Pending: Karpenter is provisioning a node (should resolve in < 60s)
   kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=30

5. CHECK if ai-worker KEDA queue is backing up
   aws sqs get-queue-attributes \
     --queue-url https://sqs.us-east-1.amazonaws.com/235270183260/ai-travel-prod-ai-jobs \
     --attribute-names ApproximateNumberOfMessages --region us-east-1
   # Large queue? KEDA will scale workers — or manually patch minReplicaCount

6. CHECK database connections
   kubectl logs -l app.kubernetes.io/name=user-service -n prod | grep "too many connections"
   # If yes: reduce replicas or upgrade RDS instance class

7. AFTER TRAFFIC SUBSIDES
   HPA and KEDA scale down automatically — verify with: kubectl get hpa,scaledobjects -n prod
   Karpenter consolidates idle nodes automatically

8. POST-INCIDENT
   Review CloudWatch metrics and Grafana dashboards
   Adjust HPA thresholds or KEDA queueLength if autoscaling reacted too slowly
```
