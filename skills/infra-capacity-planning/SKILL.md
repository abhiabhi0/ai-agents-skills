---
name: infra-capacity-planning
version: 1
description: >
  Capacity planning for backend services and infrastructure — calculate limits,
  predict saturation, plan scaling, and set SLOs. Covers Little's Law, USE method,
  goroutine/thread pool sizing, connection pool sizing, Kubernetes resource requests,
  cgroup CPU quota, and cost-aware scaling decisions. Apply when planning capacity
  for a service, sizing a deployment, setting resource limits, or deciding when
  to scale horizontally vs vertically.
triggers:
  - capacity planning
  - scaling
  - resource limits
  - GOMAXPROCS
  - connection pool
  - thread pool sizing
  - kubernetes resources
  - cpu request limit
  - memory limit
  - cgroup
  - horizontal scaling
  - vertical scaling
  - saturation
  - SLO
  - SLA
  - headroom
  - replicas
---

# Infrastructure Capacity Planning

## The USE Method (Start Here)

For every resource, check three things:

| Resource | Utilisation | Saturation | Errors |
|---|---|---|---|
| CPU | `sar -u` → `%idle` | `vmstat` → `r` column | `perf stat` → exceptions |
| Memory | `free` → used/total | `vmstat` → `si`/`so` (swap) | `dmesg` → OOM |
| Disk | `iostat` → `%util` | `iostat` → `avgqu-sz` | `dmesg` → I/O errors |
| Network | `sar -n DEV` → bandwidth | `ss` → retransmits | `ip -s link` → errors |

**Rule:** Fix saturated resources first. High utilisation without saturation is fine.

---

## Sizing Go Services

### GOMAXPROCS — CPU Core Allocation

```go
// Default: GOMAXPROCS = runtime.NumCPU() — correct for bare metal
// In containers: NumCPU() returns HOST CPUs, not the cgroup limit → wrong!
import _ "go.uber.org/automaxprocs"  // Auto-detects cgroup CPU quota — add to main.go
```

| Workload type | GOMAXPROCS guidance |
|---|---|
| CPU-bound (compression, encoding) | = number of allocated cores |
| I/O-bound (HTTP handlers, gRPC) | Default is fine; more P's = more parallelism in netpoller |
| Mixed | Default (automaxprocs handles containers correctly) |

### Max RPS — Theoretical Ceiling

Using Little's Law:
```
Max RPS = Concurrency / Avg_Latency_seconds

Example:
  GOMAXPROCS=4, avg handler latency = 20ms (0.02s)
  Max RPS = 4 / 0.02 = 200 RPS per pod

  With 5 pods: 200 × 5 = 1000 RPS theoretical max
  Apply 70% safety margin: safe operating point = 700 RPS
```

**When to add more replicas:**  
`current_rps > 0.70 × max_rps_per_pod × pod_count`

### Connection Pool Sizing

```
Pool size = (target_rps × avg_db_latency_seconds) + buffer

Example:
  1000 RPS → DB, avg DB query = 5ms (0.005s)
  Active connections = 1000 × 0.005 = 5 concurrent connections
  Pool size = 5 × 2 (headroom) = 10 connections

  For Go: database/sql
  db.SetMaxOpenConns(10)
  db.SetMaxIdleConns(5)
  db.SetConnMaxLifetime(5 * time.Minute)
```

**Signs of pool exhaustion:**
- DB query latency suddenly spikes (waiting for pool slot)
- Error: `driver: bad connection` or `context deadline exceeded` in DB logs
- `db.Stats().WaitCount` increasing in metrics

---

## Kubernetes Resource Sizing

### CPU Requests and Limits

```yaml
resources:
  requests:
    cpu: "500m"       # Scheduler uses this to place pod (guaranteed)
    memory: "256Mi"   # Guaranteed minimum
  limits:
    cpu: "2000m"      # Throttled if exceeded (causes CPU throttle, not kill)
    memory: "512Mi"   # OOMKilled if exceeded
```

**CPU throttling causes latency spikes** — set limits only if required.  
Prefer: `requests = measured p95 CPU`, `limits = 3× requests` or no limit.

```bash
# Check if container is being throttled
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled
# throttled_time_ns > 0 → CPU limit is causing latency
```

### Memory — Right-Sizing

```bash
# Get actual memory usage (not just request/limit)
kubectl top pod <pod>                  # Current usage
kubectl describe pod <pod> | grep -A5 "Limits\|Requests"

# For Go services: peak RSS during load test = memory request
# Add 30% buffer for GC overhead (Go GC doubles heap before collecting)
# memory request = peak_rss × 1.3
# memory limit = memory request × 2 (headroom before OOMKill)
```

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60    # Scale out when avg CPU > 60%
```

**Target 60% CPU** — leaves headroom for traffic spikes before new pods become ready (pod startup typically 30–120s).

---

## Scaling Decision: Horizontal vs Vertical

| Scenario | Decision | Reason |
|---|---|---|
| CPU bound, stateless service | Horizontal (more pods) | Scales linearly, no downtime |
| Memory bound, large heap | Vertical (bigger instance) | Avoids GC pressure from splitting state |
| DB connection limited | Vertical or connection proxy | DB has max connections; more pods = more connections |
| Single-threaded bottleneck | Vertical + fix the bottleneck | Adding pods doesn't help if one goroutine is the limit |
| I/O bound (network/disk) | Horizontal | I/O parallelises across pods |

---

## Setting SLOs

```
SLO = (good_requests / total_requests) × 100 ≥ target%

Typical tiers:
  Availability:  99.9% (3 nines) = 8.7 hours/year downtime
                 99.99% (4 nines) = 52 minutes/year
  Latency SLO:   p99 < 500ms on 99.9% of requests over 30-day window
  Error budget:  100% - SLO% = allowable failure rate
```

**Error budget burn rate alert:**
- If burning error budget 2× faster than normal → page on-call
- If burning 14× faster (exhausts monthly budget in 1 hour) → critical alert

---

## Load Test → Capacity Decision Workflow

```
1. Run baseline load test at 50% of expected peak
   → Record: p50, p99 latency, error rate, CPU%, memory MB

2. Ramp to 100% expected peak
   → Record same metrics + check for saturation knee

3. Ramp to 150% (spike simulation)
   → Verify: error rate <1%, p99 <2× baseline, recovery within 60s

4. Soak test at 100% for 2+ hours
   → Verify: no memory leak, no goroutine leak, no connection pool exhaustion

5. Calculate safe operating limit
   → Safe RPS = 0.70 × RPS at saturation knee

6. Set autoscaler target
   → Scale-out threshold = safe operating limit per pod
   → Min replicas = ceil(expected_baseline_rps / safe_rps_per_pod)
   → Max replicas = ceil(expected_peak_rps × 1.5 / safe_rps_per_pod)
```

---

## cgroup CPU Quota — For VMs and Containers

```bash
# Check allocated CPU quota in container
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us    # e.g. 200000 = 2 CPUs
cat /sys/fs/cgroup/cpu/cpu.cfs_period_us   # e.g. 100000 = 100ms period

# Effective CPUs = quota / period
# 200000 / 100000 = 2.0 CPUs

# Check if quota is being hit (throttling)
cat /sys/fs/cgroup/cpu/cpu.stat
# throttled_time > 0 → service is being throttled → increase CPU limit or optimise

# For Go: automaxprocs reads this and sets GOMAXPROCS correctly
import _ "go.uber.org/automaxprocs"
```

---

## KVM VM Sizing for Load Testing

```bash
# CPU: match vCPUs to load generator thread count
# Memory: loadgen_memory = (VUs × avg_request_object_size) + JVM overhead
# For JMeter: ~512MB JVM + (300 VUs × ~2MB) ≈ 1.1GB minimum
# Recommended: 4GB RAM for up to 1000 VUs

# Pin vCPUs to physical cores to avoid steal time during tests
virsh vcpupin <vm> 0 2    # vCPU 0 → physical CPU 2

# Verify no steal during test
sar -u ALL 1 5 | grep -v "^$"   # %steal should be <1% during test
# If %steal > 2% → results are unreliable (hypervisor is interfering)
```
