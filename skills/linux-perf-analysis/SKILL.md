---
name: linux-perf-analysis
version: 1
description: >
  Diagnose Linux performance bottlenecks systematically — CPU, memory, disk I/O,
  and network. Covers sar, iostat, vmstat, perf, ss, iotop, and /proc internals.
  Apply when a service is slow, resource utilisation is unexpectedly high, or
  production metrics show latency spikes and you need to find the root cause.
triggers:
  - high cpu
  - high iowait
  - cpu steal
  - slow disk
  - memory pressure
  - swap
  - context switch
  - load average
  - sar
  - iostat
  - vmstat
  - iowait
  - perf
  - linux slow
  - latency spike linux
  - process stuck
  - D state
  - kernel
---

# Linux Performance Analysis

## Triage Order — Always Follow This Sequence

1. **CPU** → 2. **Memory** → 3. **Disk I/O** → 4. **Network** → 5. **Application**

Never jump to application tuning before ruling out system-level bottlenecks.

---

## CPU

### Identify bottleneck type first

```bash
sar -u ALL 1 5          # Columns: %user %system %iowait %steal %idle
top -b -n1 -o %CPU      # Snapshot sorted by CPU
```

| Symptom | Root cause | Action |
|---|---|---|
| High `%user` | App is CPU-bound | Profile with `perf top -p <pid>` or pprof |
| High `%system` | Too many syscalls / interrupts | `strace -c -p <pid>`; check `in` column in `vmstat` |
| High `%iowait` | Waiting for disk/network I/O | → Go to Disk I/O section |
| High `%steal` (>5%) | Hypervisor throttling VM | Check KVM host CPU overcommit; pin vCPUs |
| High `%idle` but slow | Bottleneck is NOT CPU | → Go to Memory or Disk section |

### Find CPU-hungry process
```bash
ps -eo pid,ppid,%cpu,command --sort=-%cpu | head -15
perf top                           # Live top functions across all processes
perf top -p <pid>                  # Narrow to one process
```

### Context switches (high `cs` in vmstat)
```bash
vmstat 1 5                         # cs column = context switches/s
pidstat -w -p <pid> 1 5           # Per-process voluntary vs involuntary
```
- High **voluntary** switches → process is yielding (I/O wait, channel recv) — expected
- High **involuntary** switches → preempted before finishing — CPU contention

### NUMA issues (multi-socket servers)
```bash
numastat -p <pid>                  # numa_miss > 0 → remote memory access
numactl --hardware                 # Show topology
# Fix: pin process to local NUMA node
numactl --cpunodebind=0 --membind=0 ./myservice
```

---

## Memory

### Quick overview
```bash
free -h
cat /proc/meminfo | grep -E "MemAvailable|Dirty|Slab|SwapCached"
```

| Symptom | Root cause | Action |
|---|---|---|
| `MemAvailable` < 500MB | Memory pressure | Identify leaking process (see below) |
| `Dirty` growing | Writes not flushing | Check disk I/O; tune `vm.dirty_ratio` |
| `Slab` > 2GB | Kernel cache leak | `slabtop`; check dentry/inode counts |
| `si`/`so` > 0 in vmstat | Swapping actively | Latency will spike; set `vm.swappiness=0-10` |

### Find memory-hungry process
```bash
ps -eo pid,rss,command --sort=-rss | head -15  # RSS = resident RAM
pmap -x <pid> | tail -5                         # Memory map breakdown
# Watch for growth over time:
watch -n2 "cat /proc/<pid>/status | grep VmRSS"
```

### OOM risk check
```bash
dmesg | grep -i oom                             # Past OOM kills
cat /proc/<pid>/oom_score                       # Higher = killed first
# Protect critical service:
echo -1000 > /proc/<pid>/oom_score_adj
```

### Swap impact
```bash
vmstat 1 5                 # si (swap in) and so (swap out) columns
# If si/so > 0 → processes getting swapped = major page faults = high latency
sysctl -w vm.swappiness=1  # Avoid swap except emergency
```

---

## Disk I/O

### First check
```bash
iostat -xz 1 5             # -x extended stats, -z hide idle devices
```

| Column | Alert threshold | Meaning |
|---|---|---|
| `%util` | >80% | Device saturated — I/O queue is full |
| `await` | >1ms (NVMe), >10ms (SSD), >20ms (HDD) | Avg I/O latency including queue wait |
| `avgqu-sz` | >1 | Requests are queuing up |
| `r_await` vs `w_await` | Compare | Tells you if reads or writes are the problem |

### Diagnose patterns
```bash
# Find which process is doing I/O
iotop -o                           # Live, only active I/O processes

# Break down I/O per process
pidstat -d -p <pid> 1 5

# High %util + high await → disk saturated
# High await + low %util → remote storage (NFS/cloud) or single large I/O blocking queue

# I/O wait per process (D-state count)
ps aux | awk '$8 ~ /D/ {print $0}'
```

### Page cache
```bash
# Most file reads served from page cache (no disk hit)
cat /proc/meminfo | grep Cached     # Large Cached = good, I/O is cached

# Force dirty pages to flush (testing only)
sync; echo 3 > /proc/sys/vm/drop_caches
```

---

## Network

### Quick check
```bash
ss -s                               # Connection state summary
sar -n TCP 1 5                      # TCP segments, retransmits
sar -n DEV 1 5                      # Per-interface bandwidth
```

### Socket state problems

| State | Count | Problem | Fix |
|---|---|---|---|
| `TIME_WAIT` | >10k | Normal with HTTP/1.1, or port exhaustion | Enable `tcp_tw_reuse`; use keepalive |
| `CLOSE_WAIT` | Growing | App not calling `close()` after remote closes | Fix application — not a kernel issue |
| `ESTABLISHED` | Unexpected high | Connection leak or keepalive too long | Check app connection pool |
| `SYN_RECV` | High | SYN flood or slow backend | Increase `tcp_max_syn_backlog` |

```bash
ss -tnp state close-wait | head -20  # Find CLOSE_WAIT with process
ss -tnp state time-wait | wc -l      # Count TIME_WAIT
```

### Bandwidth check
```bash
sar -n DEV 1 5               # rxkB/s, txkB/s per interface
# Check against NIC capacity: 1Gbps = ~125 MB/s, 10Gbps = ~1.25 GB/s
# Near limit → NIC saturation; consider jumbo frames or bonding
```

---

## perf — Deep CPU Profiling

```bash
# Which kernel and user functions consume CPU
perf top -p <pid>

# Capture for flame graph
perf record -F 99 -p <pid> -g -- sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg

# Hardware counters (memory-bound or CPU-bound analysis)
perf stat -e cycles,instructions,cache-misses,cache-references -p <pid> sleep 5
# IPC = instructions/cycles: >2 = efficient, <1 = memory-bound

# Syscall overhead
perf trace -p <pid> 2>&1 | head -50
```

---

## Quick Reference: Symptom → Command

| Symptom | First command | Follow-up |
|---|---|---|
| Slow service, unknown cause | `top` | Sort by %CPU, then %MEM |
| High load average | `vmstat 1` | Check `r` (runnable) and `b` (blocked) |
| Disk I/O slow | `iostat -xz 1` | `iotop -o` |
| Memory growing | `ps aux --sort=-rss` | `pmap -x <pid>` |
| Network latency | `ping` + `ss -s` | `sar -n TCP 1 5` |
| High steal (VM) | `sar -u ALL 1 5` | Talk to infra — host is overloaded |
| Process in D state | `ps aux \| grep " D "` | `iotop -o` to find I/O cause |
