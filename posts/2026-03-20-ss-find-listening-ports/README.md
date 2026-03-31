# How to Find All Listening Ports with ss -l

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ss, Linux, Port, Listening, Security, Diagnostic

Description: Use ss -l to enumerate all listening TCP and UDP ports on a Linux system, with filtering by protocol, process name, and address family for security audits.

Listing all listening ports reveals what services are exposed on your server. This is critical for security audits (are unexpected services running?), firewall rule verification, and service discovery.

## Basic Listening Port Discovery

```bash
# All listening TCP sockets

ss -tl

# All listening UDP sockets
ss -ul

# Both TCP and UDP listening
ss -tul

# With port numbers (no service name resolution)
ss -tln

# The complete useful command: all listening TCP/UDP, numeric, with processes
sudo ss -tulnp
```

## Reading ss -tulnp Output

```bash
sudo ss -tulnp

# Output:
# Netid State  Recv-Q Send-Q  Local Address:Port  Process
# tcp   LISTEN     0    128   0.0.0.0:22          users:(("sshd",pid=1234))
# tcp   LISTEN     0    511   0.0.0.0:80          users:(("nginx",pid=5678))
# tcp   LISTEN     0    511   0.0.0.0:443         users:(("nginx",pid=5678))
# tcp   LISTEN     0    100  127.0.0.1:5432       users:(("postgres",pid=9012))
# udp   UNCONN     0      0   0.0.0.0:53          users:(("named",pid=1100))

# Local Address:
#   0.0.0.0:port  = listening on ALL IPv4 interfaces (externally accessible)
#   127.0.0.1:port = listening on loopback only (not externally accessible)
#   10.0.0.1:port  = listening on specific interface only
```

## Security Audit: Find Unexpected Open Ports

```bash
# List all externally accessible ports (listening on 0.0.0.0)
sudo ss -tlnp | grep '0.0.0.0:'

# Should only see ports you intentionally opened
# Unexpected entries could be:
# - Misconfigured services
# - Malware
# - Development servers left running

# Compare current ports to known-good baseline
sudo ss -tlnp | awk 'NR>1 {print $4}' | sort > /tmp/current-ports.txt
diff /tmp/baseline-ports.txt /tmp/current-ports.txt
```

## Filter by Port Range

```bash
# Find services on ports < 1024 (privileged ports)
sudo ss -tlnp | awk 'NR>1' | while read line; do
    port=$(echo $line | awk '{print $4}' | cut -d: -f2)
    if [ "$port" -lt 1024 ] 2>/dev/null; then
        echo "$line"
    fi
done

# Find services on non-standard ports
ss -tlnp | grep -v -E ':(22|80|443|25|53|3306|5432)\b'
```

## Verify Service Is Listening Before Connecting

```bash
#!/bin/bash
# wait-for-port.sh - Wait until a service starts listening

HOST="localhost"
PORT="8080"
TIMEOUT=30

echo "Waiting for $HOST:$PORT..."

for i in $(seq 1 $TIMEOUT); do
    if ss -tnl | grep -q ":${PORT} "; then
        echo "Service is listening on port $PORT"
        exit 0
    fi
    sleep 1
done

echo "Timeout: $HOST:$PORT not listening after ${TIMEOUT}s"
exit 1
```

## Check if IPv4 and IPv6 Are Both Listening

```bash
# Check if a service listens on both IPv4 and IPv6
sudo ss -tlnp | grep ':80 '

# IPv4 only: 0.0.0.0:80
# IPv6 only (may handle IPv4 too): [::]:80
# Both: two lines

# For nginx: to listen on both, use two listen directives:
# listen 80;          → IPv4
# listen [::]:80;     → IPv6
```

Regularly auditing listening ports with `ss -tulnp` is a fundamental security practice - every unexpected open port is a potential attack vector that should be investigated and closed.
