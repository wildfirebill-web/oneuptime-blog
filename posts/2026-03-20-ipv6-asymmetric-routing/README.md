# How to Troubleshoot IPv6 Asymmetric Routing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Routing, Asymmetric Routing, Troubleshooting, Multi-homing, Network Diagnostics

Description: Diagnose IPv6 asymmetric routing where packets take different paths in each direction, causing stateful firewall failures and connection issues in multi-homed environments.

## Introduction

Asymmetric routing occurs when IPv6 packets take different paths in each direction between two hosts. While the network can function, stateful firewalls and connection tracking that expect to see both directions of a flow on the same interface will break asymmetric flows. This is common in multi-homed environments with multiple uplinks.

## Detecting Asymmetric Routing

```bash
# Compare forward and reverse traceroutes

# Forward path (your host to destination)

traceroute6 -n 2001:db8::remote

# Reverse path requires running traceroute6 FROM the remote host
# Or use mtr bidirectional mode (if the remote host supports it)

# Check source address selection for the destination
ip -6 route get 2001:db8::remote
# This shows which interface and source address will be used

# Verify what the remote side would use as source
# by checking their routing table (if accessible)
```

## Step 1: Understand the Topology

```bash
# Show all IPv6 interfaces and addresses
ip -6 addr show scope global

# Show routing table
ip -6 route show

# Show which interface each route uses
ip -6 route show | awk '{print $NF, $0}' | sort

# In multi-homed host (two uplinks):
# 2001:db8:a::/64 dev eth0 - ISP A prefix
# 2001:db8:b::/64 dev eth1 - ISP B prefix
# default via fe80::gw-a dev eth0 - default uses ISP A
# Traffic from ISP B might arrive on eth1 but responses go via eth0
```

## Step 2: Diagnose with Packet Capture

```bash
# Capture on both interfaces to see asymmetric traffic
# Terminal 1: capture on eth0
sudo tcpdump -i eth0 -n "ip6 and host 2001:db8::remote"

# Terminal 2: capture on eth1
sudo tcpdump -i eth1 -n "ip6 and host 2001:db8::remote"

# If packets arrive on eth1 but responses leave on eth0 → asymmetric
# This breaks stateful firewalls
```

## Step 3: Fix with Policy-Based Routing

IPv6 policy-based routing ensures responses leave through the same interface that received the request:

```bash
# Create separate routing tables for each uplink
# Add routes to table 100 for ISP A
sudo ip -6 route add default via fe80::gw-a dev eth0 table 100

# Add routes to table 200 for ISP B
sudo ip -6 route add default via fe80::gw-b dev eth1 table 200

# Add rules: packets from ISP A addresses use table 100
sudo ip -6 rule add from 2001:db8:a::/64 table 100
# Packets from ISP B addresses use table 200
sudo ip -6 rule add from 2001:db8:b::/64 table 200

# View routing rules
ip -6 rule show
```

## Step 4: Configure rp_filter for IPv6

```bash
# Reverse Path Filtering (rp_filter) can detect asymmetric routing
# For IPv6, use ip6tables RPFILTER match
sudo ip6tables -t raw -A PREROUTING -m rpfilter --invert -j DROP

# This drops packets where the reply would not go back the same interface
# (strict mode - may be too strict for asymmetric environments)

# Loose mode alternative: only drop if no route at all
sudo ip6tables -t raw -A PREROUTING -m rpfilter --loose --invert -j DROP
```

## Step 5: Adjust Source Address Selection

```bash
# The default address selection policy (RFC 6724) may pick
# the wrong source address for outbound traffic

# View current source selection policy
cat /etc/gai.conf

# Check which source address is selected for a destination
ip -6 route get 2001:db8::remote

# Force specific source address for a destination
sudo ip -6 route add 2001:db8::remote via fe80::gw-a dev eth0 src 2001:db8:a::100

# Or use ip rule with UID-based routing for specific applications
```

## Step 6: Fix Stateful Firewall Issues

```bash
# If using conntrack with asymmetric routing:
# Allow ESTABLISHED traffic to bypass strict checks
sudo ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Or disable conntrack for specific traffic
sudo ip6tables -t raw -A PREROUTING -p tcp --dport 80 -j NOTRACK

# For load balancers with asymmetric routing:
# Use ECMP (Equal-Cost Multi-Path) and hash-based load balancing
# to ensure both directions of a flow use the same path
sudo ip -6 route add 2001:db8::/32 \
    nexthop via fe80::gw-a dev eth0 weight 1 \
    nexthop via fe80::gw-b dev eth1 weight 1
```

## Conclusion

Asymmetric IPv6 routing is common in multi-homed environments and doesn't inherently break connectivity - but it breaks stateful firewalls and connection tracking. Diagnose by capturing on all interfaces simultaneously and comparing which interface carries each flow direction. Fix with IPv6 policy-based routing to ensure responses leave through the same interface that received the request. Use `ip -6 rule add from <prefix> table <n>` for per-source routing tables, which is the standard Linux approach for multi-homed IPv6 hosts.
