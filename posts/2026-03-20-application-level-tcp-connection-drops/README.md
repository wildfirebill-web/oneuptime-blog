# How to Troubleshoot Application-Level TCP Connection Drops

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Networking, Application, Connection Drops, Troubleshooting, Linux

Description: Diagnose TCP connection drops that occur after successful establishment, examining application code, middleware timeouts, and system resource limits as causes.

## Introduction

Application-level TCP connection drops occur after the three-way handshake succeeds - the connection is established, but then unexpectedly terminates. These are harder to diagnose than connection failures because the network is working. The cause is usually application timeout logic, resource exhaustion, or middleware (load balancer, proxy) terminating idle connections.

## Identifying Application-Level Drops

```bash
# Signs of application-level drops:

# 1. "Connection reset by peer" errors in application logs
# 2. "Broken pipe" errors when writing to a socket
# 3. Connections that work sometimes but fail others
# 4. Failures correlated with specific data sizes or durations

# Capture to confirm RST packets are being sent
tcpdump -i eth0 -n 'tcp and (tcp[tcpflags] & tcp-rst != 0)' | head -20
# Source of RST tells you who is resetting: server app, client, or middlebox
```

## Common Causes and Diagnosis

### Cause 1: Application Timeout

```bash
# Application has a request timeout shorter than the actual operation
# Check application logs for timeout messages
journalctl -u myapp | grep -i "timeout\|deadline\|cancel" | tail -20

# Check if the timeout is configurable
# Python example:
# requests.get(url, timeout=5)  ← 5 second timeout
# If operation takes 6 seconds: request times out and connection drops
```

### Cause 2: Load Balancer Idle Timeout

```bash
# Load balancers drop connections idle > their configured timeout
# AWS ALB: 60 seconds default
# nginx proxy_read_timeout: 60 seconds default
# HAProxy: 50 seconds default

# Verify by testing with explicit timing:
time curl -v http://myservice/slow-endpoint 2>&1 | grep -E "time|Error"

# If failure occurs at ~60 seconds: LB timeout
# Fix: increase LB timeout OR add TCP keepalives
sysctl -w net.ipv4.tcp_keepalive_time=30
sysctl -w net.ipv4.tcp_keepalive_intvl=10
```

### Cause 3: File Descriptor Exhaustion

```bash
# When FD limit is reached, application cannot accept new connections
# And may drop existing ones

# Check current FD usage
cat /proc/$(pgrep myapp)/status | grep FDSize
ls -la /proc/$(pgrep myapp)/fd | wc -l
ulimit -n   # per-process limit

# Check for FD exhaustion errors in logs
journalctl -u myapp | grep -i "too many open files\|EMFILE"

# Increase FD limit
ulimit -n 65536

# Permanent: /etc/security/limits.conf
echo "myuser soft nofile 65536" >> /etc/security/limits.conf
echo "myuser hard nofile 65536" >> /etc/security/limits.conf
```

### Cause 4: Application Thread/Goroutine Exhaustion

```bash
# Application runs out of threads to handle new connections
# New connections are accepted but immediately dropped

# Check thread count for the process
cat /proc/$(pgrep myapp)/status | grep Threads

# For Java: thread pool exhaustion
# Check thread pool metrics via JMX or application metrics

# For Python: check if using asyncio properly
# Blocking calls in async code exhaust event loop threads
```

### Cause 5: Connection Pool Overflow

```python
# Applications with connection pools drop connections when pool is full
# Example: database connection pool exhausted

# Python with SQLAlchemy
from sqlalchemy import create_engine
engine = create_engine(
    'postgresql://user:pass@host/db',
    pool_size=10,          # Maximum connections
    max_overflow=5,        # Extra connections under load
    pool_timeout=30,       # Wait 30s for available connection
    pool_recycle=3600      # Recycle connections after 1 hour
)
# If all 15 connections busy: next request waits 30s then fails
# Fix: increase pool_size or diagnose why connections aren't being returned
```

## Systematic Diagnosis

```bash
# 1. When does it fail? (timing pattern)
# 2. What does the RST look like? (source IP, sequence context)
tcpdump -i eth0 -n -v 'tcp[tcpflags] & tcp-rst != 0' -c 10

# 3. Check application metrics
# - Active connection count
# - Error rate
# - Response time distribution

# 4. Check system resource limits
ulimit -a
ss -tn | wc -l                # Current connection count
cat /proc/sys/fs/file-nr      # System-wide FD usage
```

## Conclusion

Application-level TCP drops require investigating beyond the network layer. The RST source identifies the culprit: application RST indicates timeout or resource exhaustion in the app; middlebox RST indicates proxy/LB timeout. Enable TCP keepalives for idle connection retention, check resource limits (FDs, threads, connection pools), and correlate failure timing with configured timeout values.
