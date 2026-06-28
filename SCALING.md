# Scaling Guide

## Overview

The AI-Travel-Planner platform uses three complementary layers of scaling to handle variable
traffic loads while maintaining cost efficiency:

| Layer | Technology | What Scales | Trigger |
|-------|-----------|-------------|---------|
| **Pod (horizontal)** | Kubernetes HPA | Replica count per service | CPU utilization |
| **Node (horizontal)** | Cluster Autoscaler | EC2 nodes in EKS node group | Pending pod scheduling |
| **Database (vertical)** | RDS instance resize | CPU/memory/IOPS per DB instance | Manual, during maintenance window |

The general scaling philosophy:
1. **Scale pods first** — cheapest, fastest, zero-downtime
2. **Scale nodes second** — needed when pods can't fit on existing nodes
3. **Scale database last** — requires careful planning (brief failover for Multi-AZ)

---

## Horizontal Pod Autoscaling (HPA)

### Current HPA Configuration

| Service | Min Replicas | Max Replicas | CPU Target | Notes |
|---------|-------------|-------------|-----------|-------|
| `ai-service` | 2 | 5 | 70% | Bedrock calls are slow (2–5s), I/O-bound |
| `frontend` | 2 | 10 | 60% CPU + 70% Memory | nginx static serving |

> **Note on `ai-service`:** Bedrock Nova Pro responses typically take 2–5 seconds. The service
> is I/O-bound during that wait. CPU utilization is a reasonable proxy — it rises as the thread
> pool fills up with in-flight requests. Keep an eye on latency during peak loads.

`user-service`, `travel-service`, `utility-service`, and `ai-worker` run at a fixed
`replicaCount: 2` (worker: 1) defined in their `values.yaml`. Scale them manually
with `kubectl scale` or by updating `replicaCount` in the GitOps repo.

### How HPA Works — Timeline

```
T+0s    Pod CPU exceeds threshold (70%)
T+15s   HPA queries metrics-server (15-second scrape interval)
T+15s   HPA calculation: current=85%, target=70%, ratio=1.21
T+15s   HPA decision: ceil(2 × 1.21) = 3 replicas (was 2)
T+30s   New pod scheduled, Kubernetes finds a node with capacity
T+35s   New pod starts, pulls image from ECR (cached locally → fast)
T+50s   New pod passes readiness probe
T+50s   New pod starts receiving traffic, load distributed across 3 pods
--- Load decreases ---
T+350s  CPU drops below threshold for 5+ minutes (scale-down stabilization)
T+350s  HPA reduces to 2 replicas (scale down is deliberately slower)
T+360s  Old pod gracefully terminated (SIGTERM, 30s grace period)
```

### HPA Manifest Reference

```yaml
# Example: helm-charts/user-service/templates/hpa.yaml
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
          periodSeconds: 60            # Or double the count, whichever is larger
      selectPolicy: Max
```

### Tuning HPA

**`targetCPUUtilizationPercentage` guidelines:**

| Service Type | Recommended Target | Reason |
|-------------|-------------------|--------|
| CPU-intensive (compute) | 60–70% | Leave headroom for sudden spikes |
| I/O-intensive (DB calls) | 70–80% | CPU stays low even under load |
| Memory-intensive (caching) | 70% CPU | Memory leaks are harder to recover |
| Long-running requests (AI) | 70% | Requests queue up quickly at high load |

**When to increase `maxReplicas`:**

If `kubectl get hpa` shows replicas constantly at the max and CPU is still above target,
increase `maxReplicas` in the service's `values.yaml` and push to GitOps. ArgoCD will apply
the change within 3 minutes.

### Monitoring HPA

```bash
# View HPA status and current metrics
kubectl get hpa -n prod

# Detailed HPA status (shows scaling events)
kubectl describe hpa user-service -n prod
# Look for: "ScalingActive", "AbleToScale", "ScalingLimited"
# And the "Events:" section for recent scaling decisions

# Watch HPA in real-time
watch kubectl get hpa -n prod

# View current pod resource usage
kubectl top pods -n prod --sort-by=cpu
kubectl top pods -n prod --sort-by=memory

# Historical HPA events
kubectl get events -n prod \
  --field-selector reason=SuccessfulRescale \
  --sort-by='.lastTimestamp'
```

---

## Cluster Autoscaler (Node Scaling)

### How Cluster Autoscaler Works

```
[New Pod created]
      │
      ▼ (scheduler)
Pod is Pending (can't fit on any existing node)
      │
      ▼ (cluster-autoscaler, runs every 10s)
CA scans pending pods and calculates required node count
      │
      ▼
CA calls AWS Auto Scaling API: SetDesiredCapacity(current + 1)
      │
      ▼ (~2–3 minutes)
New EC2 instance launches, bootstraps, joins EKS cluster
      │
      ▼
Pending pod is scheduled on new node
      │
      ▼ (pod starts, traffic handled)

[Scale-down: when node utilization < 50% for 10 minutes]
CA drains node pods (respects PodDisruptionBudgets)
CA calls AWS: SetDesiredCapacity(current - 1)
EC2 instance terminates (~5 minutes total)
```

### Current Node Group Configuration

| Parameter | Value |
|-----------|-------|
| Instance type | t3.medium (2 vCPU, 4 GB RAM) |
| AMI | Amazon Linux 2 (managed by EKS) |
| Min nodes | 3 |
| Desired nodes | 3 |
| Max nodes | 6 |
| Capacity type | On-Demand |
| Scale-up trigger | Any pod in `Pending` state due to insufficient resources |
| Scale-down threshold | Node request utilization < 50% for 10 minutes |
| Scale-down delay after add | 10 minutes (prevents oscillation) |
| Expander | `least-waste` (chooses the node group that wastes the least resources) |

### Cluster Autoscaler Configuration (`platform-cluster-autoscaler`)

The Cluster Autoscaler is deployed via Helm chart `cluster-autoscaler` v9.46.6
into `kube-system` and managed by ArgoCD. It uses IRSA for AWS API access.

Key settings:
- **Auto-discovery**: Finds the node group via ASG tags `k8s.io/cluster-autoscaler/enabled=true`
  and `k8s.io/cluster-autoscaler/ai-travel-prod=owned`
- **IRSA role**: `arn:aws:iam::235270183260:role/ai-travel-prod-cluster-autoscaler`
- **Permissions**: `autoscaling:SetDesiredCapacity`, `autoscaling:TerminateInstanceInAutoScalingGroup`
  (scoped to tagged ASGs only), plus read-only describe permissions

### Pod Disruption Budgets (PDB)

PDBs prevent the Cluster Autoscaler from draining a node if it would violate the minimum
available pod count. All services have PDBs configured:

```yaml
# At least 1 pod must remain available during node drain
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

### When to Upgrade Node Size

| Symptom | Evidence | Recommended Action |
|---------|---------|-------------------|
| Pods OOMKilled frequently | `kubectl describe pod` shows `OOMKilled` reason | Increase pod memory limits or upgrade to t3.large |
| CPU throttling constant | `kubectl top pods` shows CPU near limit | Increase CPU limits or upgrade node type |
| 6 nodes not enough | Cluster Autoscaler logs: "max node group size reached" | Increase `node_group_max_size` in Terraform |
| Scale-up takes too long | Pod stuck `Pending` >5 minutes | Pre-scale or add a second node group |

### Terraform Changes for Node Scaling

```hcl
# PlanMyjourney-Terraform/environments/prod.tfvars

# Option A: Increase maximum node count (scale out)
node_group_min_size     = 3
node_group_max_size     = 10      # was 6
node_group_desired_size = 3       # Cluster Autoscaler manages this

# Option B: Upgrade node type (scale up)
node_instance_types     = ["t3.large"]   # was ["t3.medium"]
# t3.large: 2 vCPU, 8 GB RAM — doubles memory per node
# t3.xlarge: 4 vCPU, 16 GB RAM — for compute-heavy workloads
```

**Apply the change:**

```bash
cd PlanMyjourney-Terraform
terraform plan -var-file="environments/prod.tfvars" -target=module.eks.aws_eks_node_group.workers
# Review: node group update will trigger rolling replacement (~5 min per node)

terraform apply -var-file="environments/prod.tfvars" -target=module.eks.aws_eks_node_group.workers
```

> **Zero-downtime:** Rolling node replacement respects PodDisruptionBudgets and drains nodes
> one at a time. All services maintain at least 1 running pod throughout the update.

### Cluster Autoscaler Monitoring

```bash
# Check Cluster Autoscaler logs
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=cluster-autoscaler \
  --tail=100

# Key log messages to watch for:
# "Scale up: setting group ... to X" — scaling up
# "Scale down: removing node ..." — scaling down
# "max node group size reached" — need to increase node_group_max_size

# Check current node count
kubectl get nodes

# Check pending pods (scale-up trigger)
kubectl get pods -A | grep Pending

# Watch node auto-scaling events
kubectl get events -n kube-system \
  --field-selector reason=TriggeredScaleUp \
  --sort-by='.lastTimestamp'
```

---

## Database Scaling (RDS)

### Current RDS Configuration

| Parameter | Value |
|-----------|-------|
| Instance class | db.t3.micro |
| vCPU | 2 |
| RAM | 1 GB |
| Max connections | 87 |
| Storage | 50 GB gp3 |
| Storage auto-scaling | Up to 100 GB |
| IOPS | 3,000 baseline (gp3) |
| Throughput | 125 MB/s baseline (gp3) |

### Vertical Scaling (Upgrading Instance Class)

Vertical scaling requires a **maintenance window** (Multi-AZ failover = ~20 seconds of
application-level retry needed, but typically transparent):

**Upgrade Path:**
```
db.t3.micro (2 vCPU, 1 GB, 87 max conn)
    ↓
db.t3.small (2 vCPU, 2 GB, 193 max conn)
    ↓
db.t3.medium (2 vCPU, 4 GB, 415 max conn)
    ↓
db.t3.large (2 vCPU, 8 GB, 874 max conn)
    ↓
db.r6g.large (2 vCPU, 16 GB, Graviton, 1000+ max conn)
```

**Perform the Upgrade:**

```bash
# 1. Update Terraform variable
# PlanMyjourney-Terraform/environments/prod.tfvars
db_instance_class = "db.t3.small"

# 2. Check what will change
cd PlanMyjourney-Terraform
terraform plan -var-file="environments/prod.tfvars" -target=module.rds

# 3. Schedule during low-traffic window (the failover takes ~20s)
terraform apply -var-file="environments/prod.tfvars" -target=module.rds

# 4. Monitor the modification
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region us-east-1 \
  --query "DBInstances[0].{Status: DBInstanceStatus, Class: DBInstanceClass}"

# 5. Verify applications reconnect (SQLAlchemy pool_pre_ping handles this)
kubectl get pods -n prod | grep -v Running
# A brief connection reset is normal; pods should not crash
```

### Connection Pool Tuning

As you scale horizontally (more pods), connection pool settings become critical:

**Current formula:**
```
Max DB connections:    87  (db.t3.micro)
Connections per pod:   10 (pool_size) + 20 (max_overflow) = 30 peak
Max safe pods:         87 / 30 = 2.9 → max 2 pods safely
```

This means with `db.t3.micro`, you **cannot run more than 2 replicas** per DB-connected service
without risking "too many connections" errors.

**Solutions:**

| Solution | Cost | Downtime | Scalability |
|----------|------|----------|------------|
| Upgrade to db.t3.small | +$22/month | ~20s failover | 4–6 pods per service |
| Reduce pool_size (5 instead of 10) | Free | None | 2× more pods |
| Add PgBouncer connection pooler | Free (add pod) | None | 100s of pods |
| RDS Proxy | +$20/month | None | Unlimited |

---

## Load Testing

### Install k6

```bash
# macOS
brew install k6

# Windows (via scoop)
scoop install k6

# Linux
sudo gpg -k
sudo gpg --no-default-keyring \
  --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt update && sudo apt install k6
```

### Load Test Scripts

**Basic health endpoint stress test:**

```javascript
// tests/load/health-check.js
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 10 },   // Ramp up to 10 users
    { duration: '60s', target: 50 },   // Ramp up to 50 users
    { duration: '60s', target: 50 },   // Hold at 50 users
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
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

**AI Service load test (Bedrock calls):**

```javascript
// tests/load/ai-service.js
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  vus: 5,            // Only 5 concurrent users (Bedrock is slow)
  duration: '120s',
  thresholds: {
    http_req_duration: ['p(95)<6000'],  // Bedrock p95 < 6 seconds
    http_req_failed: ['rate<0.05'],     // Allow 5% errors (Bedrock limits)
  },
};

export default function () {
  const payload = JSON.stringify({
    message: "Plan a 3-day trip to Goa with a budget of $500"
  });
  const response = http.post(
    'https://api.invest-iq.online/api/ai/plan',
    payload,
    { headers: { 'Content-Type': 'application/json' } }
  );
  check(response, {
    'status is 200': (r) => r.status === 200,
    'has travel plan': (r) => r.json('plan') !== undefined,
  });
  sleep(5);  // Be respectful of Bedrock rate limits
}
```

### Observing Scaling During Load Test

```bash
# In a separate terminal: watch pods scale
watch kubectl get hpa,pods -n prod

# Watch node scaling
watch kubectl get nodes

# Watch resource utilization
watch kubectl top pods -n prod
```

---

## Performance Benchmarks

Target latencies under normal load (< 50 concurrent users):

| Service | p50 Latency | p95 Latency | p99 Latency | Max RPS | Notes |
|---------|------------|------------|------------|---------|-------|
| `frontend` | < 50ms | < 200ms | < 500ms | 500 RPS | nginx static serving |
| `user-service` | < 100ms | < 500ms | < 1,000ms | 200 RPS | DB-bound |
| `travel-service` | < 100ms | < 400ms | < 800ms | 150 RPS | DB-bound |
| `ai-service` | < 2,000ms | < 5,000ms | < 10,000ms | 10 RPS | Bedrock latency dominates |
| `utility-service` | < 200ms | < 1,000ms | < 2,000ms | 100 RPS | External API latency |

---

## Scaling Runbook: Traffic Spike Response

Follow this runbook when receiving alerts about high latency or pod overload:

```
1. CHECK current state
   kubectl get hpa -n prod          # Is HPA already scaling?
   kubectl top pods -n prod         # Which service is overloaded?
   kubectl get nodes                 # Are nodes available?

2. IDENTIFY the overloaded service
   # Look for: CPU > 80% in `kubectl top pods` output

3. MANUAL SCALE (if HPA is slow to react)
   kubectl scale deployment user-service --replicas=4 -n prod

4. CHECK if nodes are the bottleneck
   kubectl get pods -n prod | grep Pending
   # If Pending: Cluster Autoscaler is adding a node (wait 3–5 min)
   # Check CA logs: kubectl logs -n kube-system -l app.kubernetes.io/name=cluster-autoscaler --tail=50

5. CHECK database connections
   kubectl logs -l app.kubernetes.io/name=user-service -n prod | grep "too many connections"
   # If yes: reduce replicas or add PgBouncer

6. AFTER TRAFFIC SUBSIDES
   # HPA will scale down automatically after 5 minutes
   # Verify with: kubectl get hpa -n prod

7. POST-INCIDENT
   # Review CloudWatch metrics and adjust HPA thresholds if needed
   # Consider increasing node_group_max_size if CA hit the ceiling
```
