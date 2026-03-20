# How to Configure Connection Tracking Limits in iptables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: iptables, conntrack, Linux, Security, Connection Tracking, DDoS

Description: Configure iptables connection tracking table size and per-IP connection limits to prevent conntrack table exhaustion from connection floods.

The iptables connection tracking (conntrack) table records every active connection. When it fills up, new connections are dropped and the server becomes unreachable. Tuning conntrack limits and using connlimit rules prevents exhaustion.

## Understanding the conntrack Table

```bash
# View current conntrack table usage
sudo conntrack -L | wc -l
# or
cat /proc/sys/net/netfilter/nf_conntrack_count

# View maximum table size
cat /proc/sys/net/netfilter/nf_conntrack_max

# View table utilization
echo "Current: $(cat /proc/sys/net/netfilter/nf_conntrack_count)"
echo "Maximum: $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
```

## Increasing the conntrack Table Size

The default max (typically 65536) is too small for high-traffic servers:

```bash
# Increase conntrack table to 1 million entries
sudo sysctl -w net.netfilter.nf_conntrack_max=1048576

# Also increase the hash table size (should be max/4)
sudo sysctl -w net.netfilter.nf_conntrack_buckets=262144

# Make permanent
echo "net.netfilter.nf_conntrack_max = 1048576" | sudo tee -a /etc/sysctl.conf
echo "net.netfilter.nf_conntrack_buckets = 262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Reducing conntrack Timeouts

Old entries consuming table space can be evicted faster with shorter timeouts:

```bash
# View current timeout values
sysctl -a 2>/dev/null | grep conntrack | grep timeout

# Reduce TCP established timeout (default 432000s = 5 days!)
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400

# Reduce TIME_WAIT timeout
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30

# Reduce UDP timeout
sudo sysctl -w net.netfilter.nf_conntrack_udp_timeout=30

# Make permanent in /etc/sysctl.conf
cat << EOF | sudo tee -a /etc/sysctl.conf
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_udp_timeout = 30
EOF
```

## Limit Connections Per IP with connlimit

Use iptables connlimit module to cap connections from each source:

```bash
# Reject if a single IP has more than 50 concurrent connections
sudo iptables -A INPUT -p tcp -m connlimit --connlimit-above 50 -j REJECT

# Limit HTTP connections per IP
sudo iptables -A INPUT -p tcp --dport 80 \
  -m connlimit --connlimit-above 20 -j REJECT

# Limit SSH connections per IP (max 3 concurrent)
sudo iptables -A INPUT -p tcp --dport 22 \
  -m connlimit --connlimit-above 3 -j REJECT

# Connlimit by subnet (limit per /24)
sudo iptables -A INPUT -p tcp --dport 80 \
  -m connlimit --connlimit-above 100 --connlimit-mask 24 -j REJECT
```

## Monitor and Alert on Table Fullness

```bash
#!/bin/bash
# check-conntrack.sh — alert when conntrack is near full

MAX=$(cat /proc/sys/net/netfilter/nf_conntrack_max)
CURRENT=$(cat /proc/sys/net/netfilter/nf_conntrack_count)
PERCENT=$((CURRENT * 100 / MAX))

echo "conntrack: $CURRENT / $MAX ($PERCENT% full)"

if [ "$PERCENT" -gt 80 ]; then
    echo "WARNING: conntrack table is ${PERCENT}% full!"
    # Optionally flush old entries
    # sudo conntrack -F
fi
```

## View conntrack Entries

```bash
# List all tracked connections
sudo conntrack -L

# Filter by protocol
sudo conntrack -L -p tcp

# Filter by state
sudo conntrack -L | grep ESTABLISHED | wc -l
sudo conntrack -L | grep TIME_WAIT | wc -l

# Find IPs with most connections
sudo conntrack -L -p tcp | grep -oP 'src=\S+' | sort | uniq -c | sort -rn | head -20

# Delete a specific connection
sudo conntrack -D -s 1.2.3.4
```

## Drop Invalid Connections

Conntrack marks packets that don't match any valid state as INVALID. Drop them:

```bash
# Drop packets with invalid conntrack state
sudo iptables -A INPUT -m state --state INVALID -j DROP
sudo iptables -A FORWARD -m state --state INVALID -j DROP

# This also prevents some TCP state machine attacks
```

Keeping conntrack healthy is essential — a full conntrack table silently drops all new connections without any error, making it look like the network is completely down.
