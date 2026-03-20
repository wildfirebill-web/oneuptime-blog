# How to Optimize IPv6 Routing Table Lookups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Routing, Linux, Performance, Kernel, FIB

Description: Optimize Linux IPv6 routing table lookup performance through route cache tuning, FIB trie configuration, and kernel parameter adjustments for large routing tables.

## Introduction

IPv6 routing table lookups use the Linux FIB (Forwarding Information Base) trie structure. On hosts with thousands of routes (e.g., BGP full tables), lookup performance can degrade without proper tuning. This guide covers kernel settings and architectural strategies to keep forwarding fast.

## Step 1: Understand the IPv6 Route Cache

Linux removed the IPv6 route cache in kernel 3.12+. Routes are now looked up directly in the FIB trie for each packet. Check your current route table size:

```bash
# Count IPv6 routes in the main table
ip -6 route show | wc -l

# Show routing table statistics
ip -6 route show table all | wc -l

# Check the FIB trie statistics
cat /proc/net/fib_triestat 2>/dev/null || \
  cat /proc/net/rt6_stats
```

## Step 2: Tune FIB Hash Table Size

```bash
# View current FIB table size limits
sysctl net.ipv6.route.max_size

# Increase maximum IPv6 routing table size
# Default is 4096, increase for BGP full-table hosts
echo "net.ipv6.route.max_size = 2147483647" | \
  sudo tee -a /etc/sysctl.d/99-ipv6-routing.conf

# Adjust garbage collection thresholds
echo "net.ipv6.route.gc_thresh = 1024" | \
  sudo tee -a /etc/sysctl.d/99-ipv6-routing.conf
echo "net.ipv6.route.gc_min_interval_ms = 500" | \
  sudo tee -a /etc/sysctl.d/99-ipv6-routing.conf
echo "net.ipv6.route.gc_timeout = 60" | \
  sudo tee -a /etc/sysctl.d/99-ipv6-routing.conf

sudo sysctl -p /etc/sysctl.d/99-ipv6-routing.conf
```

## Step 3: Use Multiple Routing Tables for Policy Routing

Splitting routes across multiple tables reduces per-table size and lookup times.

```bash
# Add a custom routing table entry to /etc/iproute2/rt_tables
echo "100 ipv6_primary" >> /etc/iproute2/rt_tables

# Add a route to the custom table
ip -6 route add 2001:db8::/32 via 2001:db8::1 table 100

# Add a rule to use the custom table for specific sources
ip -6 rule add from 2001:db8:1::/48 lookup 100

# List all routing rules
ip -6 rule show
```

## Step 4: Enable ECMP for Load-Balanced Routing

Equal-Cost Multi-Path routing distributes traffic across multiple next-hops, improving both throughput and lookup efficiency.

```bash
# Add ECMP route with multiple next-hops
ip -6 route add 2001:db8:100::/48 \
  nexthop via 2001:db8::1 dev eth0 weight 1 \
  nexthop via 2001:db8::2 dev eth1 weight 1

# Verify ECMP route
ip -6 route show 2001:db8:100::/48

# Enable per-flow ECMP hashing (default in modern kernels)
sysctl net.ipv6.fib_multipath_hash_policy
# 0 = L3 (src+dst IP), 1 = L4 (adds ports)
echo "net.ipv6.fib_multipath_hash_policy = 1" | \
  sudo tee -a /etc/sysctl.d/99-ipv6-routing.conf
```

## Step 5: Profile Routing Lookup Performance

```bash
# Use perf to measure time spent in IPv6 FIB lookups
sudo perf stat -e cache-misses,cache-references \
  -a sleep 10 2>&1 | grep cache

# Measure packet forwarding rate
# (requires two hosts and iperf3 at line rate)
iperf3 -6 -c 2001:db8::1 -t 60 -P 8 --format m

# Check for route lookup errors
ip -6 -s route show | grep -i error
netstat -s6 | grep -i "route\|forward"
```

## Common Optimization Patterns

```bash
# Summarize more-specific routes into aggregates when possible
# Instead of 1000 /128 routes, use fewer /48 or /64 aggregates
ip -6 route flush cache 2>/dev/null || true

# Disable IPv6 on interfaces that don't need it (reduces table size)
sysctl net.ipv6.conf.eth1.disable_ipv6=1
```

## Conclusion

IPv6 routing table performance in Linux is governed by the FIB trie and GC thresholds. Increasing `max_size`, using policy routing tables, enabling ECMP, and aggregating routes where possible are the most impactful optimizations. Monitor forwarding rates and lookup latency with OneUptime's network performance dashboards.
