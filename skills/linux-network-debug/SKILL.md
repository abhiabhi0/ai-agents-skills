---
name: linux-network-debug
version: 1
description: >
  Debug Linux network issues end-to-end: socket leaks, connection exhaustion,
  TCP state problems, packet loss, latency, DNS failures, and kernel TCP tuning.
  Apply when services have connection timeouts, CLOSE_WAIT/TIME_WAIT buildup,
  port exhaustion, network latency spikes, or packet drops.
triggers:
  - connection refused
  - connection timeout
  - CLOSE_WAIT
  - TIME_WAIT
  - port exhaustion
  - network latency
  - packet loss
  - tcp
  - socket
  - ss command
  - netstat
  - dns failure
  - connection reset
  - epoll
  - FD limit
  - too many open files
  - connection leak
  - network slow
---

# Linux Network Debugging

## Decision Tree

```
Service slow / failing?
  ├─ Connection refused       → Is the service listening? Port open? FD limit hit?
  ├─ Connection timeout       → Firewall? Network route? Server backlog full?
  ├─ Slow responses           → Bandwidth? Latency? DNS? Keep-alive missing?
  ├─ CLOSE_WAIT accumulating  → Application bug — not closing sockets
  ├─ TIME_WAIT high           → Normal or port exhaustion — tune tcp_tw_reuse
  └─ Random resets            → tcp_keepalive? MTU mismatch? Load balancer timeout?
```

---

## Socket State Analysis

```bash
ss -s                          # Summary of all socket states
ss -tnp                        # All TCP sockets with process names
ss -tnp state established      # Only active connections
```

### CLOSE_WAIT — Always an Application Bug
```bash
ss -tnp state close-wait       # List them with process
# CLOSE_WAIT = remote peer sent FIN, but local app hasn't called close()
# Fix: find the code path that opens connections and ensure close/defer close is called
# In Go: always defer resp.Body.Close() for HTTP; defer conn.Close() for raw TCP
```

### TIME_WAIT — Often Normal, Can Cause Port Exhaustion
```bash
ss -tnp state time-wait | wc -l   # Count
cat /proc/sys/net/ipv4/ip_local_port_range  # Available ephemeral ports
# If TIME_WAIT count ≈ port range → port exhaustion
# Fix 1: Enable reuse
sysctl -w net.ipv4.tcp_tw_reuse=1
# Fix 2: Widen port range
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
# Fix 3: Use HTTP keepalive (reduces connection churn)
```

### Too Many Open Files (FD limit)
```bash
# Error: "too many open files" or connection refused at high concurrency
ulimit -n                               # Current per-process FD limit (often 1024)
cat /proc/sys/fs/file-max               # System-wide limit
lsof -p <pid> | wc -l                   # How many FDs this process has open

# Fix — permanent (add to /etc/security/limits.conf):
# * soft nofile 1048576
# * hard nofile 1048576
# Or in systemd unit file:
# [Service]
# LimitNOFILE=1048576
```

---

## Connection Refused / Not Listening

```bash
# Is the service actually listening?
ss -tlnp | grep <port>           # t=tcp l=listening n=numeric p=process

# Is firewall blocking?
iptables -L -n -v | grep <port>
nft list ruleset | grep <port>

# Is the listen backlog full? (connections dropped before accept())
ss -tlnp | grep <port>           # Recv-Q = pending in backlog, Send-Q = max backlog
# If Recv-Q = Send-Q → backlog is full → increase:
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
# Also increase in your app: net.Listen + set socket SO_BACKLOG
```

---

## Latency Diagnosis

```bash
# Basic RTT
ping -c 20 <host>                # Look for jitter and packet loss

# Traceroute with timing
traceroute -n <host>             # Find slow hop
mtr -n <host>                    # Continuous traceroute with stats

# TCP connection time (excluding app processing)
curl -o /dev/null -s -w "connect: %{time_connect}s  TTFB: %{time_starttransfer}s  total: %{time_total}s
" http://<host>/path

# DNS resolution time
time dig <hostname>
# High DNS time → add caching (systemd-resolved, dnsmasq) or use IP directly
```

### Simulating network conditions for testing (tc netem)
```bash
# Add 50ms latency to outgoing packets on eth0
tc qdisc add dev eth0 root netem delay 50ms

# Latency + jitter (±10ms, normal distribution)
tc qdisc add dev eth0 root netem delay 50ms 10ms distribution normal

# Packet loss 1%
tc qdisc add dev eth0 root netem loss 1%

# Bandwidth throttle to 10Mbit/s
tc qdisc add dev eth0 root tbf rate 10mbit burst 32kbit latency 400ms

# Combine latency + loss
tc qdisc add dev eth0 root netem delay 50ms loss 0.5%

# Check current rules
tc qdisc show dev eth0

# Remove all
tc qdisc del dev eth0 root
```

---

## TCP Keepalive — Detect Dead Connections

```bash
# Check current values
sysctl net.ipv4.tcp_keepalive_time     # Default 7200s (2 hours!) — too long
sysctl net.ipv4.tcp_keepalive_intvl    # Default 75s
sysctl net.ipv4.tcp_keepalive_probes   # Default 9

# Tune for faster dead connection detection
sysctl -w net.ipv4.tcp_keepalive_time=60      # Start probing after 60s idle
sysctl -w net.ipv4.tcp_keepalive_intvl=10     # Probe every 10s
sysctl -w net.ipv4.tcp_keepalive_probes=3     # 3 failures = dead

# In Go: set on socket directly
import "syscall"
syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, syscall.SO_KEEPALIVE, 1)
# Or use net.Dialer{KeepAlive: 30 * time.Second}
```

---

## Packet Drops

```bash
# NIC-level drops
ip -s link show eth0
# RX errors/dropped/overrun > 0 → NIC ring buffer full or driver issue
# Fix: increase ring buffer
ethtool -G eth0 rx 4096

# TCP retransmits (congestion or loss)
sar -n TCP 1 5           # retrans/s column
ss -ti                   # Per-socket TCP info including retransmits

# Conntrack table full (drops silently)
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
# If count ≈ max → increase max or reduce conntrack timeout:
sysctl -w net.netfilter.nf_conntrack_max=524288
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=300
```

---

## Bandwidth Saturation

```bash
sar -n DEV 1 5          # rxkB/s txkB/s per interface
# 1Gbps NIC → ~125 MB/s max; 10Gbps → ~1.25 GB/s
# Near limit? → check for single process hogging bandwidth:
iftop -n                # Real-time per-connection bandwidth
nethogs                 # Per-process bandwidth
```

---

## Quick Fixes Cheatsheet

| Problem | Fix |
|---|---|
| Port exhaustion | `tcp_tw_reuse=1`; widen `ip_local_port_range` |
| CLOSE_WAIT leak | Fix app — add `defer conn.Close()` |
| Backlog dropped | Increase `somaxconn` + `tcp_max_syn_backlog` |
| FD limit hit | `LimitNOFILE=1048576` in systemd unit |
| Dead connections not detected | Lower `tcp_keepalive_time` to 60s |
| DNS slow | Add local caching; use `/etc/hosts` for internal services |
| Packet drops at NIC | Increase ring buffer with `ethtool -G` |
| Conntrack full | Increase `nf_conntrack_max`; reduce TCP timeout |
