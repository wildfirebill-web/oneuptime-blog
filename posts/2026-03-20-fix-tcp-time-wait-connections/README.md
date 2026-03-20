# How to Fix Too Many TCP Connections in TIME_WAIT State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, TIME_WAIT, Networking, Performance, Kernel

Description: Understand why TIME_WAIT connections accumulate and apply kernel tuning to reduce their impact on port exhaustion and connection performance.

## Introduction

TIME_WAIT is a normal TCP state that every short-lived connection passes through after closing. It persists for 2×MSL (typically 60 seconds) to ensure late packets don't confuse new connections. In high-traffic services handling thousands of short connections per second, the accumulation of TIME_WAIT sockets can exhaust local port ranges or use significant memory.

## Diagnosing the Problem

```bash
# Count TIME_WAIT connections

ss -tn state time-wait | wc -l

# See which remote addresses have the most TIME_WAIT
ss -tn state time-wait | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Check if you're running out of local ports
ss -tn state time-wait | awk '{print $4}' | cut -d: -f2 | sort -n | head -5
# If local port numbers are very high or repeating, you may have port exhaustion
```

## Understanding Why TIME_WAIT Happens

```text
TIME_WAIT is created by the ACTIVE CLOSER - the side that sends the first FIN.

- HTTP/1.0 server closes connection after each request → server has TIME_WAIT
- HTTP client with connection pooling → client has TIME_WAIT when pool shrinks
- Microservice calling downstream API → calling service has TIME_WAIT
```

## Fix 1: Enable tcp_tw_reuse

Allows reusing TIME_WAIT sockets for outbound connections to different servers. Safe and recommended:

```bash
# Allow reuse of TIME_WAIT sockets for new outbound connections
sysctl -w net.ipv4.tcp_tw_reuse=1

# Persist
echo "net.ipv4.tcp_tw_reuse=1" >> /etc/sysctl.conf
sysctl -p

# Note: Only helps for client (connecting) side, not server (listening) side
```

## Fix 2: Use HTTP Keep-Alive / Connection Pooling

The best fix: don't create so many short-lived connections in the first place:

```python
# Python requests with connection pooling (keeps connections alive)
import requests

# Without pooling (creates TIME_WAIT for each request)
# for url in urls:
#     requests.get(url)   # Bad: new connection each time

# With connection pooling (reuses connections)
session = requests.Session()
session.mount('http://', requests.adapters.HTTPAdapter(
    pool_connections=10,
    pool_maxsize=20
))

for url in urls:
    session.get(url)   # Good: connection reused
```

## Fix 3: Expand the Local Port Range

More available ports means TIME_WAIT is less likely to cause port exhaustion:

```bash
# View current range (default: 32768-60999 = 28232 ports)
sysctl net.ipv4.ip_local_port_range

# Expand the range
sysctl -w net.ipv4.ip_local_port_range="10000 65535"
# Now 55535 ports available
```

## Fix 4: Reduce TIME_WAIT Duration (Not Recommended)

```bash
# tcp_fin_timeout affects FIN_WAIT2 state, not directly TIME_WAIT
# But reducing it can help in some scenarios
sysctl net.ipv4.tcp_fin_timeout
# Default: 60 seconds

# Reduce to 30 seconds
sysctl -w net.ipv4.tcp_fin_timeout=30
```

## Monitoring Improvement

```bash
# Watch TIME_WAIT count over time
watch -n 5 "ss -tn state time-wait | wc -l"

# After applying fixes, TIME_WAIT count should stabilize below port range exhaustion
# A healthy system with short-lived connections may have thousands of TIME_WAIT
# - this is normal as long as it doesn't exceed ~50,000
```

## Conclusion

TIME_WAIT accumulation is a symptom of many short-lived connections, not a bug. The best fix is connection pooling and HTTP keep-alive to reduce connection churn. `tcp_tw_reuse=1` is a safe kernel-level mitigation for outbound connections. Expanding the local port range provides headroom. Avoid `tcp_tw_recycle` - it was removed in Linux 4.12 due to breaking NAT scenarios.
