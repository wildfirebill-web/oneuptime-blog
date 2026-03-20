# How to Display TCP Socket States Using ss

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ss, TCP, Linux, Socket States, Networking, Diagnostics

Description: Use ss to filter and display TCP socket states including ESTABLISHED, TIME_WAIT, CLOSE_WAIT, and LISTEN to diagnose connection problems and resource leaks.

TCP connection states reveal the lifecycle stage of each connection. Monitoring states with `ss` helps diagnose resource exhaustion (too many TIME_WAIT), connection leaks (stuck CLOSE_WAIT), and unusual traffic patterns.

## TCP State Reference

```text
State          Description
-------------  -------------------------------------------------
LISTEN         Waiting for incoming connection
SYN_SENT       Sent SYN, waiting for SYN-ACK
SYN_RECV       Received SYN, sent SYN-ACK
ESTABLISHED    Active, bidirectional connection
FIN_WAIT1      Sent FIN, waiting for ACK
FIN_WAIT2      Received ACK of FIN, waiting for remote FIN
CLOSE_WAIT     Remote closed, local hasn't closed yet
CLOSING        Both sides closing simultaneously
LAST_ACK       Sent FIN after CLOSE_WAIT, waiting for ACK
TIME_WAIT      Connection ended, holding to catch late packets
CLOSED         Connection terminated
```

## Filter by Specific State

```bash
# Show established connections

ss -tn state established

# Show listening sockets
ss -tn state listening

# Show TIME_WAIT connections (normal after connection closes)
ss -tn state time-wait

# Show CLOSE_WAIT (potential connection leak in application)
ss -tn state close-wait

# Show SYN_RECV (SYN flood indicator if high count)
ss -tn state syn-recv
```

## Count Connections by State

```bash
# Count all states
ss -ta | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# Expected healthy output:
#   45 ESTAB      (established connections)
#   12 TIME-WAIT  (recently closed - normal)
#    3 LISTEN     (services)

# Concerning patterns:
#  500+ TIME-WAIT → high connection turnover (web servers under load)
#   50+ CLOSE-WAIT → application not closing connections (memory/socket leak)
#  100+ SYN-RECV  → possible SYN flood attack
```

## Diagnose High TIME_WAIT Count

```bash
# Count TIME_WAIT connections
ss -tn state time-wait | wc -l

# If > 1000, identify the ports causing accumulation
ss -tn state time-wait | awk '{print $4}' | cut -d: -f2 | sort | uniq -c | sort -rn

# Fix options:
# 1. Allow TIME_WAIT reuse (for outbound connections)
sudo sysctl -w net.ipv4.tcp_tw_reuse=1

# 2. Increase ephemeral port range
sudo sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# 3. Reduce TIME_WAIT duration (use with caution)
# Default is 2*MSL = 60 seconds; cannot easily reduce in modern kernels
```

## Diagnose CLOSE_WAIT (Application Bug)

```bash
# CLOSE_WAIT means: remote side closed, but LOCAL application hasn't called close()
# This indicates a bug in your application

# Find which process has CLOSE_WAIT sockets
sudo ss -tnp state close-wait

# Look for pattern: always the same application, increasing count over time
# Fix: fix the application code to properly close connections
# Temporary workaround: restart the application

# Monitor growth over time
watch -n 5 'ss -tn state close-wait | wc -l'
```

## Watch All States in Real Time

```bash
#!/bin/bash
# tcp-states.sh - Show TCP state summary

while true; do
    echo "=== TCP States $(date '+%H:%M:%S') ==="
    ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
    echo "---"
    sleep 5
done
```

## Find Connections in Non-ESTABLISHED States

```bash
# Find any connection that's NOT in ESTABLISHED or LISTEN state
# These are connections in transition - useful for debugging handshake issues
ss -tan | grep -v -E "^(ESTAB|LISTEN|State)" | head -20
```

Understanding TCP states directly from `ss` gives you real-time visibility into your application's network behavior - high counts of unusual states reliably signal bugs or attacks.
