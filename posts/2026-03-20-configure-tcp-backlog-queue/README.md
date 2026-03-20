# How to Configure TCP Backlog Queue Size on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, Networking, Performance, Backlog, Kernel

Description: Configure the TCP connection backlog queue to handle connection bursts without dropping SYNs, tuning both kernel parameters and application listen() calls.

## Introduction

The TCP backlog queue is a buffer that holds incoming connection requests before the application accepts them. On Linux, this is actually two queues: the SYN queue (incomplete connections awaiting the final ACK) and the accept queue (completed connections ready for the application). When these queues fill up, incoming SYNs are silently dropped, causing connection timeouts from the client's perspective.

## The Two Queues

```
SYN queue (tcp_max_syn_backlog):
  - Holds half-open connections (after SYN, before ACK)
  - Managed by kernel
  - Default: 128-1024 depending on system

Accept queue (somaxconn + listen() backlog):
  - Holds fully established connections waiting for accept()
  - Managed by application + kernel
  - Default: min(listen_backlog, net.core.somaxconn)
```

## Checking Queue Fill Status

```bash
# Show listen sockets with queue statistics
ss -tlnp
# Column meanings:
# Recv-Q: Current size of accept queue (connections waiting for app to accept)
# Send-Q: Maximum size of accept queue (= backlog setting)

# If Recv-Q approaches Send-Q: application is not accepting fast enough
# tcp LISTEN 0 128  0.0.0.0:8080  ...
#              ^ ^   Recv-Q=0, Send-Q=128 (backlog = 128)

# Check drop statistics
netstat -s | grep "SYNs to LISTEN sockets dropped"
# or
ss -s | grep "TCP"
```

## Tuning the SYN Queue

```bash
# Increase SYN queue size (half-open connections)
sysctl net.ipv4.tcp_max_syn_backlog
# Default: 128-512

sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# Persist
echo "net.ipv4.tcp_max_syn_backlog=4096" >> /etc/sysctl.conf
```

## Tuning the Accept Queue

```bash
# Step 1: Increase system maximum for accept queue
sysctl net.core.somaxconn
# Default: 128

sysctl -w net.core.somaxconn=1024

# Persist
echo "net.core.somaxconn=1024" >> /etc/sysctl.conf
sysctl -p
```

The application also needs to pass a large backlog to `listen()`:

```python
import socket

# Python server with proper backlog configuration
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 8080))

# Pass large backlog — OS uses min(backlog, net.core.somaxconn)
server.listen(1024)
print("Listening with backlog=1024")
```

## Nginx and Application Server Configuration

```nginx
# nginx: the accept queue size is controlled by nginx's listen backlog
server {
    listen 80 backlog=4096;  # Increase from default 511
}
```

```bash
# systemd socket activation: set ListenBacklog
# /etc/systemd/system/myapp.socket
[Socket]
ListenStream=8080
Backlog=4096
```

## Verifying the Configuration

```bash
# Check effective backlog after changes
ss -tlnp | grep "8080"
# Send-Q should reflect the new backlog value

# Stress test: generate a burst of connections
for i in $(seq 1 1000); do
    nc -z 10.20.0.5 8080 &
done
wait

# Check if connections were accepted without drops
ss -s | grep TCP
netstat -s | grep "SYNs to LISTEN sockets dropped"
# Should be 0 if backlog is adequate
```

## Conclusion

Backlog queue configuration requires changes at two levels: the kernel (somaxconn, tcp_max_syn_backlog) and the application (listen() call). The effective backlog is `min(listen_backlog, net.core.somaxconn)`. For high-traffic services handling bursts, 1024-4096 is appropriate. Monitor Recv-Q in `ss -tlnp` to detect when queues are filling up.
