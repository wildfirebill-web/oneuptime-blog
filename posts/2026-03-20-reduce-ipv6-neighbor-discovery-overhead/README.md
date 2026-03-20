# How to Reduce IPv6 Neighbor Discovery Overhead

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Neighbor Discovery, NDP, Linux, Performance, Kernel

Description: Reduce IPv6 Neighbor Discovery Protocol (NDP) overhead on Linux by tuning neighbor cache parameters, suppressing unnecessary solicitations, and optimizing multicast behavior.

## Introduction

IPv6 Neighbor Discovery Protocol (NDP) replaces ARP and handles address resolution, router discovery, and duplicate address detection. On busy networks or hosts with many neighbors, NDP can generate significant overhead. This guide covers tuning strategies to minimize its impact.

## Step 1: Tune the Neighbor Cache

The neighbor (ND) cache stores IPv6-to-MAC mappings. A small or aggressive GC policy causes frequent re-solicitation.

```bash
# /etc/sysctl.d/99-ndp-tuning.conf

# Minimum number of entries before GC starts (increase for busy servers)
net.ipv6.neigh.default.gc_thresh1 = 1024

# Soft limit — GC runs more aggressively above this
net.ipv6.neigh.default.gc_thresh2 = 4096

# Hard limit — entries are forcibly removed above this
net.ipv6.neigh.default.gc_thresh3 = 8192

# How long (seconds) before a neighbor entry is considered stale
net.ipv6.neigh.default.gc_stale_time = 60

# Interval (ms) between GC runs
net.ipv6.neigh.default.gc_interval = 30000

# How long to keep a confirmed neighbor (base reachability time ms)
net.ipv6.neigh.default.base_reachable_time_ms = 30000

# Delay (ms) between Neighbor Solicitations
net.ipv6.neigh.default.delay_first_probe_time = 5000

# Number of NS retransmissions before marking unreachable
net.ipv6.neigh.default.ucast_solicit = 3
net.ipv6.neigh.default.mcast_solicit = 3
```

Apply:

```bash
sudo sysctl -p /etc/sysctl.d/99-ndp-tuning.conf
```

## Step 2: Disable Unnecessary Router Solicitations

On servers with static network configuration, router solicitations are wasteful.

```bash
# Disable router solicitations on a specific interface
sysctl net.ipv6.conf.eth0.router_solicitations=0

# Accept RA messages only if needed (set to 0 for static config)
sysctl net.ipv6.conf.eth0.accept_ra=0

# Persist settings
cat >> /etc/sysctl.d/99-ndp-tuning.conf <<EOF
net.ipv6.conf.eth0.router_solicitations = 0
net.ipv6.conf.eth0.accept_ra = 0
EOF
```

## Step 3: Reduce Duplicate Address Detection Probes

DAD sends Neighbor Solicitations before using a new address. On trusted internal networks, reduce or disable DAD.

```bash
# Reduce DAD transmissions (default is 1, set to 0 to disable)
# 0 disables DAD — only appropriate on trusted, isolated networks
net.ipv6.conf.eth0.dad_transmits = 0

# Or reduce to a single transmission
net.ipv6.conf.eth0.dad_transmits = 1
```

## Step 4: Suppress Multicast Listener Reports on Links with No Multicast Routers

```bash
# Disable MLD (Multicast Listener Discovery) reports on non-multicast links
sysctl net.ipv6.conf.eth0.mc_forwarding=0

# Check current MLD state
ip -6 maddr show dev eth0
```

## Step 5: Monitor NDP Traffic

```bash
# Capture NDP packets with tcpdump
sudo tcpdump -i eth0 -n \
  "icmp6 and (ip6[40] == 133 or ip6[40] == 134 or ip6[40] == 135 or ip6[40] == 136)"
# Types: 133=RS, 134=RA, 135=NS, 136=NA

# Count NDP message rates per second
sudo tcpdump -i eth0 -n -l \
  "icmp6 and (ip6[40] == 135 or ip6[40] == 136)" 2>/dev/null | \
  pv -l -i 1 > /dev/null

# Show current neighbor cache entries
ip -6 neigh show dev eth0

# Show neighbor cache statistics
ip -6 -s neigh show dev eth0
```

## Step 6: Use Static Neighbor Entries for Critical Hosts

For gateways and frequently-accessed hosts, static ND entries eliminate solicitation overhead entirely.

```bash
# Add a permanent neighbor entry
ip -6 neigh add 2001:db8::1 lladdr aa:bb:cc:dd:ee:ff dev eth0

# Verify
ip -6 neigh show dev eth0 | grep "PERMANENT"
```

## Conclusion

NDP overhead is manageable with proper cache sizing, reduced DAD transmissions, and static entries for stable infrastructure. On large-scale deployments, these optimizations can significantly reduce control-plane traffic. Use OneUptime to alert on network errors that might indicate NDP failures.
