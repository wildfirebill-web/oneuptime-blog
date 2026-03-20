# How to Configure Asymmetric Routing with Reverse Path Filtering

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Routing, Asymmetric Routing, Reverse Path Filtering, rp_filter, Networking, Security

Description: Configure asymmetric routing on Linux by adjusting reverse path filtering (rp_filter) settings to allow traffic that arrives and departs via different interfaces.

## Introduction

Asymmetric routing occurs when packets from A to B take a different path than packets from B to A. Linux's reverse path filtering (rp_filter) blocks packets that arrive on an interface different from the one the kernel would use to send a reply. For legitimate asymmetric routing scenarios, you must relax or disable rp_filter.

## Check Current rp_filter Settings

```bash
# Check rp_filter for all interfaces
sysctl -a | grep rp_filter

# Check specific interface
sysctl net.ipv4.conf.eth0.rp_filter
sysctl net.ipv4.conf.all.rp_filter

# Values:
# 0 = No filtering
# 1 = Strict mode (packet must return via same interface)
# 2 = Loose mode (packet source must be in any routing table)
```

## Disable rp_filter for Asymmetric Routing

```bash
# Set loose mode (value 2) for a specific interface
sysctl -w net.ipv4.conf.eth0.rp_filter=2
sysctl -w net.ipv4.conf.eth1.rp_filter=2

# Or disable entirely (value 0) — less safe
sysctl -w net.ipv4.conf.eth0.rp_filter=0

# The 'all' key takes effect for newly created interfaces
sysctl -w net.ipv4.conf.all.rp_filter=2
```

## Make rp_filter Changes Persistent

```bash
# Add to /etc/sysctl.conf or /etc/sysctl.d/99-routing.conf
cat >> /etc/sysctl.d/99-routing.conf << 'EOF'
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.eth0.rp_filter = 2
net.ipv4.conf.eth1.rp_filter = 2
EOF

# Apply immediately
sysctl -p /etc/sysctl.d/99-routing.conf
```

## Asymmetric Routing Example

Traffic from Client A comes in via eth0, reply goes out via eth1 (multi-homed server):

```bash
# Configure routing tables for each ISP
ip route add default via 10.0.0.1 table 100
ip route add default via 10.0.1.1 table 200

# Policy rules for correct reply routing
ip rule add from 10.0.0.5 table 100
ip rule add from 10.0.1.5 table 200

# Set loose rp_filter to allow asymmetric paths
sysctl -w net.ipv4.conf.eth0.rp_filter=2
sysctl -w net.ipv4.conf.eth1.rp_filter=2
```

## Diagnose rp_filter Drops

```bash
# Count packets dropped by rp_filter
ip -s route show table local | grep -A2 broadcast

# Use conntrack statistics
conntrack -S

# Check interface drop counters
ip -s link show eth0 | grep -A2 RX
```

## rp_filter Modes Summary

| Mode | Behavior | Use Case |
|---|---|---|
| 0 | No filter | Full asymmetric routing |
| 1 | Strict | Single-homed hosts |
| 2 | Loose | Multi-homed, asymmetric |

## Conclusion

Reverse path filtering protects against spoofed source addresses but interferes with legitimate asymmetric routing. Set `rp_filter=2` (loose mode) on multi-homed interfaces to allow asymmetric paths while maintaining some protection. Use policy routing rules alongside rp_filter adjustments to ensure correct reply routing.
