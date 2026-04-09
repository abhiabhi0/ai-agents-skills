---
name: load-test-interpreter
version: 1
description: >
  Interpret load test results from JMeter/BlazeMeter and correlate with Kong API
  Gateway metrics, system resources, and Go service behaviour. Covers latency
  percentiles, error patterns, saturation detection, OTT/video streaming metrics,
  and capacity planning decisions. Apply when running or analysing load tests,
  reading JMeter summary reports, diagnosing performance under load, or planning
  capacity for a service.
triggers:
  - load test
  - JMeter
  - BlazeMeter
  - p99
  - p95
  - latency
  - throughput
  - rps
  - tps
  - error rate
  - kong
  - upstream
  - kong active connections
  - capacity planning
  - apdex
  - virtual users
  - saturation
  - OTT
  - streaming performance
  - TSB
  - TTFB
---

# Load Test Interpretation

## Reading JMeter Summary Report

| Column | Meaning | Action if bad |
|---|---|---|
| `Average` | Mean response time — **do not use alone** | Always check p99 alongside |
| `90th pct` | p90 latency — 90% of users faster than this | SLI target for most services |
| `95th pct` | p95 — only 5% slower | Standard SLA for APIs |
| `99th pct` | p99 — only 1% slower (tail latency) | Critical: p99 > 3× p50 = problem |
| `Error%` | % failed requests (5xx + timeouts) | >0.1% under target load = investigate |
| `Throughput` | Actual RPS achieved | Compare to target RPS |
| `Avg Bytes` | Avg response size | Large = bandwidth bottleneck possible |

**p99 > 3× p50** → GC pause, lock contention, or noisy neighbour  
**p99 >> p95** → Rare but catastrophic events (OOM pressure, cold caches, GC STW)  
**Average ≈ p99** → Distribution is uniform (good); or all responses are slow (bad)

---

## Diagnosing Error% Spikes Under Load

```
Error% rises as VUs increase?
  ├─ 5xx from Kong         → Kong upstream health check: curl localhost:8001/upstreams/<name>/health
  ├─ 5xx from service      → Check Go service logs: goroutine count, GC pressure, OOM
  ├─ Connection refused    → FD limit hit (ulimit -n) or backlog full (ss -tlnp)
  ├─ Timeout errors        → Service latency > client timeout — NOT the same as 5xx
  └─ 429 Too Many Requests → Kong rate limiting triggered — check Kong rate-limit plugin config
```

### At what VU count did errors start?
```
Errors start at N VUs → that's your saturation point
Below N VUs:  service handles load with acceptable latency
Above N VUs:  queue builds up → latency rises → timeouts cascade → errors
```

---

## Kong API Gateway Metrics

### Status Endpoint
```bash
curl http://localhost:8001/status
# {
#   "server": {
#     "connections_active": 142,    ← total open connections (all states)
#     "connections_reading": 2,     ← reading client request header
#     "connections_writing": 35,    ← writing response to client
#     "connections_waiting": 105    ← keep-alive idle connections
#   }
# }
```

| Metric | Normal | Problem |
|---|---|---|
| `connections_waiting` high | Yes — these are idle keepalive | Not a problem |
| `connections_writing` = `connections_active` | No idle connections | Service is fully busy |
| `connections_active` growing without bound | — | Upstream too slow; connections piling up |

### Kong Prometheus Metrics (key)
```bash
# Latency histogram for p99
histogram_quantile(0.99, rate(kong_latency_ms_bucket{type="request"}[1m]))

# Error rate
rate(kong_http_requests_total{code=~"5.."}[1m]) / rate(kong_http_requests_total[1m])

# Upstream health
kong_upstream_target_health{state="healthy"}   # 1 = healthy, 0 = unhealthy
```

### Kong Upstream Health Check
```bash
curl http://localhost:8001/upstreams/<name>/health
# Look for: data[].health = "UNHEALTHY"
# If unhealthy during load → service is returning errors → Kong circuit-breaker
# → Check service logs at the exact time errors started
```

---

## Latency Percentiles — What Each Reveals

| Percentile | What it represents | Typical SLA target |
|---|---|---|
| p50 (median) | Typical user experience | Informational |
| p90 | 9 in 10 users are faster | Non-critical APIs |
| p95 | 19 in 20 users are faster | Standard API SLA |
| p99 | 99 in 100 users are faster | Critical path SLA |
| p99.9 | 999 in 1000 users are faster | Payment, auth, streaming start |

**If p99 spikes but p50 is fine** → rare slow event: GC pause, lock contention, cache miss  
**If p50 and p99 both rise proportionally** → system-wide saturation  
**If only p99.9 spikes** → outlier requests; check for specific endpoints or payload sizes

---

## Saturation Point & Capacity Planning

### Little's Law
```
Active Concurrency = Throughput (RPS) × Latency (seconds)

Example:
  Target: 1000 RPS, avg latency 50ms (0.05s)
  Concurrency needed = 1000 × 0.05 = 50 concurrent requests

  If GOMAXPROCS=8, and each request uses 1 goroutine:
  Headroom = 8 physical CPUs → queue builds when concurrency > ~8 × (CPU util factor)
```

### Finding Saturation Point in Test Results
```
Plot: RPS (x-axis) vs p99 Latency (y-axis)
  ├─ Flat section          → service handling load comfortably
  ├─ Knee in curve         → saturation point — throughput stops scaling
  └─ Steep rise after knee → queue building, latency exploding

Safe operating RPS = 70% of saturation point RPS
```

---

## Load Test Phases — What to Measure When

```
Ramp-up (ignore metrics)  |  Steady State (measure here)  |  Spike  |  Soak
     VUs: 0→target              VUs: constant at target      2×VUs   Hours at target
```

| Phase | What to validate |
|---|---|
| Steady state (30min+) | p95/p99 latency, error rate, throughput |
| Spike | Autoscaling response, circuit breaker, p99 spike magnitude |
| Soak (2-4 hours) | Memory leak (Go heap growth), connection pool exhaustion, goroutine leak |

**Soak test signals of memory leak in Go:**
- `HeapAlloc` in pprof growing monotonically during steady load
- NumGC increasing but `HeapInuse` not decreasing after GC
- Goroutine count growing → `curl localhost:6060/debug/pprof/goroutine?debug=1`

---

## OTT / Video Streaming Metrics

| Metric | Meaning | Alert When |
|---|---|---|
| TTFB | Time to first byte — startup latency | >500ms for live stream start |
| Segment download time | Time to fetch one `.ts`/`.mp4` chunk | >segment duration (e.g. >2s for 2s segment) |
| Manifest request latency | `.m3u8` / `.mpd` playlist fetch | >200ms — fetched very frequently |
| CDN hit rate | % requests served by CDN vs origin | <90% = origin getting hammered |
| Bitrate ladder selection | Which ABR quality tier selected | Mostly low tiers = bandwidth bottleneck |
| TSB offset | How far behind live edge | Growing offset = segment generation slow |

**If segment download time > segment duration** → clients will stall and rebuffer  
**If manifest p99 > 500ms** → ABR player will delay quality switches  
**If CDN hit rate drops during load test** → CDN cache eviction under load or TTL too short

---

## Apdex Score

```
Apdex = (Satisfied + 0.5 × Tolerating) / Total

Satisfied  = response < T       (T = threshold, typically 500ms)
Tolerating = T ≤ response < 4T
Frustrated = response ≥ 4T or error

Score:  >0.94 = Excellent  |  0.85–0.94 = Good  |  0.70–0.84 = Fair  |  <0.70 = Poor
```

---

## Correlation Checklist During Load Test

When latency or errors rise, check these in parallel:

```bash
# Go service
curl localhost:6060/debug/vars | jq '{goroutines,gc_pause_total_ns:.memstats.PauseTotalNs,heap_mb:(.memstats.HeapAlloc/1048576)}'

# Kong
curl localhost:8001/status
curl localhost:8001/upstreams/<name>/health

# OS
sar -u 1 3          # CPU
iostat -xz 1 3      # Disk
ss -s               # Connections
vmstat 1 3          # Memory + swap
```
