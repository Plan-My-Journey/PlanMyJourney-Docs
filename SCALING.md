# Scaling Guide

## Overview

The AI-Travel-Planner platform uses three complementary layers of scaling to handle variable
traffic loads while maintaining cost efficiency:

| Layer | Technology | What Scales | Trigger |
|-------|-----------|-------------|---------|
| **Pod (horizontal)** | Kubernetes HPA | Replica count per service | CPU/memory utilization |
| **Node (horizontal)** | Cluster Autoscaler | EC2 nodes in EKS node group | Pending pod scheduling |
| **Database (vertical)** | RDS instance resize | CPU/memory/IOPS per DB instance | Manual, during maintenance window |

The general scaling philosophy:
1. **Scale pods first** — cheapest, fastest, zero-downtime
2. **Scale nodes second** — needed when pods can't fit on existing nodes
3. **Scale database last** — requires careful planning (brief failover for Multi-AZ)

---

## Horizontal Pod Autoscaling (HPA)

### Current HPA Configuration

| Service | Min Replicas | Max Replicas | CPU Target | Memory Target | Notes |
|---------|-------------|-------------|-----------|--------------|-------|
| `frontend` | 2 | 8 | 70% | — | SSR workload, scales well |
| `user-service` | 2 | 6 | 70% | 80% | DB-bound, check connections |
| `travel-service` | 2 | 6 | 70% | 80% | DB-bound |
| `ai-service` | 2 | 4 | 60% | 80% | Bedrock calls are slow (2–5s) |
| `utility-service` | 2 | 5 | 70% | 80% | External API rate limits apply |

> **Note on `ai-service`:** Bedrock Nova Pro responses typically take 2–5 seconds. The service
> is I/O-bound, not CPU-bound. The lower CPU target (60%) prevents overloading during latency
> spikes when Bedrock is slow.

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
# infrastructure/helm-charts/user-service/templates/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service
  namespace: ai-travel
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilizationPercentage: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilizationPercentage: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale up immediately
      policies:
        - type: Pods
          value: 2                        # Add max 2 pods per 60 seconds
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 minutes before scaling down
      policies:
        - type: Pods
          value: 1                        # Remove max 1 pod per 120 seconds
          periodSeconds: 120
```

### Tuning HPA

**`targetCPUUtilizationPercentage` guidelines:**

| Service Type | Recommended Target | Reason |
|-------------|-------------------|--------|
| CPU-intensive (compute) | 60–70% | Leave headroom for sudden spikes |
| I/O-intensive (DB calls) | 70–80% | CPU stays low even under load |
| Memory-intensive (caching) | 70% CPU, 80% memory | Memory leaks are harder to recover |
| Long-running requests (AI) | 60% | Requests queue up quickly at high load |

**When to use custom metrics (requests-per-second):**

CPU utilization is a lagging indicator for `ai-service` because Bedrock requests are mostly
waiting (I/O), not using CPU. Consider using RPS-based scaling with KEDA (Kubernetes Event
Driven Autoscaler) if you find HPA is not scaling fast enough:

```yaml
# KEDA ScaledObject for ai-service (RPS-based, requires Prometheus)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ai-service-keda
  namespace: ai-travel
spec:
  scaleTargetRef:
    name: ai-service
  minReplicaCount: 2
  maxReplicaCount: 4
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: http_requests_per_second
        query: rate(http_requests_total{service="ai-service"}[2m])
        threshold: "10"   # Scale when >10 RPS per pod
```

### Monitoring HPA

```bash
# View HPA status and current metrics
kubectl get hpa -n ai-travel

# Detailed HPA status (shows scaling events)
kubectl describe hpa user-service -n ai-travel
# Look for: "ScalingActive", "AbleToScale", "ScalingLimited"
# And the "Events:" section for recent scaling decisions

# Watch HPA in real-time
watch kubectl get hpa -n ai-travel

# View current pod resource usage
kubectl top pods -n ai-travel --sort-by=cpu
kubectl top pods -n ai-travel --sort-by=memory

# Historical HPA events
kubectl get events -n ai-travel \
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
CA scans pending pods and calculates required node size
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
| Min nodes | 2 |
| Max nodes | 5 |
| Scale-up trigger | Any pod in `Pending` state due to insufficient resources |
| Scale-down threshold | Node request utilization < 50% for 10 minutes |
| Scale-down delay after add | 10 minutes (prevents oscillation) |
| Max scale-up per iteration | 1 node |

### Pod Disruption Budgets (PDB)

PDBs prevent the Cluster Autoscaler from draining a node if it would violate the minimum
available pod count. All services have PDBs configured:

```yaml
# At least 1 pod must remain available during node drain
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
  namespace: ai-travel
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: user-service
```

### When to Upgrade Node Size

| Symptom | Evidence | Recommended Action |
|---------|---------|-------------------|
| Pods OOMKilled frequently | `kubectl describe pod` shows `OOMKilled` reason | Increase pod memory limits or upgrade to t3.large |
| CPU throttling constant | `kubectl top pods` shows CPU near limit | Increase CPU limits or upgrade node type |
| 5 nodes not enough | Cluster Autoscaler logs: "max nodes reached" | Increase `max_size` in Terraform |
| Scale-up takes too long | Pod stuck `Pending` >5 minutes | Add a second node group or pre-scale |

### Terraform Changes for Node Scaling

```hcl
# infrastructure/terraform/environments/prod.tfvars

# Option A: Increase maximum node count (scale out)
node_group_min_size     = 2
node_group_max_size     = 10      # was 5
node_group_desired_size = 2       # autoscaler manages this

# Option B: Upgrade node type (scale up)
node_instance_types     = ["t3.large"]   # was ["t3.medium"]
# t3.large: 2 vCPU, 8 GB RAM — doubles memory per node
# t3.xlarge: 4 vCPU, 16 GB RAM — for compute-heavy workloads

# Option C: Mixed instance policy (Spot + On-Demand for cost savings)
# Enable in eks module: use_mixed_instances_policy = true
# on_demand_percentage_above_base_capacity = 50  # 50% on-demand, 50% spot
```

**Apply the change:**

```bash
cd infrastructure/terraform
terraform plan -var-file="environments/prod.tfvars" -target=module.eks.aws_eks_node_group.main
# Review: node group update will trigger rolling replacement (~5 min per node)

terraform apply -var-file="environments/prod.tfvars" -target=module.eks.aws_eks_node_group.main
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
# "max node group size reached" — need to increase max_size

# Check current node count
kubectl get nodes | wc -l

# Check pending pods (scale-up trigger)
kubectl get pods -A | grep Pending
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
# infrastructure/terraform/environments/prod.tfvars
rds_instance_class = "db.t3.small"

# 2. Check what will change
cd infrastructure/terraform
terraform plan -var-file="environments/prod.tfvars" -target=module.rds

# 3. Schedule during low-traffic window (the failover takes ~20s)
# Apply with --apply-immediately for urgent cases, or wait for maintenance window
terraform apply -var-file="environments/prod.tfvars" -target=module.rds

# 4. Monitor the modification
aws rds describe-db-instances \
  --db-instance-identifier ai-travel-prod \
  --region ap-south-1 \
  --query "DBInstances[0].{Status: DBInstanceStatus, Class: DBInstanceClass}"

# 5. Verify applications reconnect (SQLAlchemy pool_pre_ping handles this)
kubectl get pods -n ai-travel | grep -v Running
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

**PgBouncer deployment (recommended for >10 replicas per service):**

```yaml
# Add to Helm chart: templates/pgbouncer.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: ai-travel
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: pgbouncer
          image: bitnami/pgbouncer:latest
          env:
            - name: POSTGRESQL_HOST
              value: "<rds-endpoint>"
            - name: POSTGRESQL_PORT
              value: "5432"
            - name: PGBOUNCER_POOL_MODE
              value: "transaction"         # Best for microservices
            - name: PGBOUNCER_MAX_CLIENT_CONN
              value: "500"
            - name: PGBOUNCER_DEFAULT_POOL_SIZE
              value: "20"
```

### Adding a Read Replica

For read-heavy workloads (analytics, reporting), a read replica offloads SELECT queries from
the primary instance without requiring an upgrade.

```hcl
# infrastructure/terraform/modules/rds/read-replica.tf
resource "aws_db_instance" "read_replica" {
  identifier             = "ai-travel-prod-read"
  replicate_source_db    = aws_db_instance.main.identifier
  instance_class         = "db.t3.micro"
  publicly_accessible    = false
  skip_final_snapshot    = true
  deletion_protection    = false

  tags = {
    Name    = "ai-travel-prod-read-replica"
    Project = "ai-travel"
    Role    = "read-replica"
  }
}

output "rds_read_endpoint" {
  value = aws_db_instance.read_replica.endpoint
}
```

```python
# In the application: use separate read/write connections
write_engine = create_engine(os.environ["DATABASE_URL"])           # Primary
read_engine  = create_engine(os.environ["DATABASE_READ_URL"])      # Read replica

# Route read-only queries to the replica:
with read_engine.connect() as conn:
    results = conn.execute(text("SELECT * FROM trips WHERE user_id = :uid"), {"uid": user_id})
```

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
  const response = http.get('https://api.aitravel.com/health');
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  sleep(1);
}
```

**Run the test:**

```bash
k6 run tests/load/health-check.js

# With output to InfluxDB for Grafana visualization
k6 run --out influxdb=http://localhost:8086/k6 tests/load/health-check.js
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
    'https://api.aitravel.com/ai/plan',
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
watch kubectl get hpa,pods -n ai-travel

# Watch node scaling
watch kubectl get nodes

# Watch resource utilization
watch kubectl top pods -n ai-travel
```

---

## Performance Benchmarks

Target latencies under normal load (< 50 concurrent users):

| Service | p50 Latency | p95 Latency | p99 Latency | Max RPS | Notes |
|---------|------------|------------|------------|---------|-------|
| `frontend` | < 50ms | < 200ms | < 500ms | 500 RPS | Next.js SSR with CDN |
| `user-service` | < 100ms | < 500ms | < 1,000ms | 200 RPS | DB-bound |
| `travel-service` | < 100ms | < 400ms | < 800ms | 150 RPS | DB-bound |
| `ai-service` | < 2,000ms | < 5,000ms | < 10,000ms | 10 RPS | Bedrock latency dominates |
| `utility-service` | < 200ms | < 1,000ms | < 2,000ms | 100 RPS | External API latency |

**Bedrock latency breakdown (Nova Pro, ap-south-1):**
- Network to Bedrock VPC endpoint: < 5ms (stays in VPC)
- Model inference time: 1,500–4,000ms (varies by prompt length)
- Response streaming (if enabled): first token in ~500ms

### Identifying Performance Bottlenecks

```bash
# Check which pods are using the most CPU
kubectl top pods -n ai-travel --sort-by=cpu

# Check which pods are using the most memory
kubectl top pods -n ai-travel --sort-by=memory

# Check RDS slow query log
aws rds describe-db-log-files \
  --db-instance-identifier ai-travel-prod \
  --filename-contains "slowquery" \
  --region ap-south-1

# Enable slow query log if not already on (adds ~5% overhead)
aws rds modify-db-parameter-group \
  --db-parameter-group-name ai-travel-prod-pg15 \
  --parameters "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate"
# Now all queries >1000ms are logged to CloudWatch
```

---

## Scaling Runbook: Traffic Spike Response

Follow this runbook when receiving alerts about high latency or pod overload:

```
1. CHECK current state
   kubectl get hpa -n ai-travel     # Is HPA already scaling?
   kubectl top pods -n ai-travel    # Which service is overloaded?
   kubectl get nodes                # Are nodes available?

2. IDENTIFY the overloaded service
   # Look for: CPU > 80% in `kubectl top pods` output

3. MANUAL SCALE (if HPA is slow to react)
   kubectl scale deployment user-service --replicas=4 -n ai-travel

4. CHECK if nodes are the bottleneck
   kubectl get pods -n ai-travel | grep Pending
   # If Pending: cluster autoscaler is adding a node (wait 3–5 min)

5. CHECK database connections
   kubectl logs -l app=user-service -n ai-travel | grep "too many connections"
   # If yes: reduce replicas or add PgBouncer

6. AFTER TRAFFIC SUBSIDES
   # HPA will scale down automatically after 5 minutes
   # Verify with: kubectl get hpa -n ai-travel

7. POST-INCIDENT
   # Review CloudWatch metrics and adjust HPA thresholds if needed
   # Consider Reserved Instances if peak traffic is recurring
```
