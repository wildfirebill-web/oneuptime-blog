# How to Troubleshoot Asymmetric Routing Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Routing, Troubleshooting, Firewall, IPv4, Asymmetric

Description: Diagnose and fix asymmetric routing problems where forward and return traffic take different paths, causing dropped connections and stateful firewall failures.

## Introduction

Asymmetric routing occurs when the forward path (client to server) and the return path (server to client) traverse different network devices. While the network layer tolerates this, stateful firewalls and connection-tracking systems expect to see both directions of a flow — and will drop packets when they see only one side.

## Detecting Asymmetric Routing

Use traceroute in both directions to compare paths:

```bash
# From host A (192.168.1.10), trace to host B (10.20.0.10)
traceroute 10.20.0.10

# From host B, trace back to host A
traceroute 192.168.1.10

# If the hops differ, routing is asymmetric
```

On Linux, check which interface traffic leaves through:

```bash
# Show which interface and gateway the kernel will use for a destination
ip route get 10.20.0.10

# Check the reverse path
ip route get 192.168.1.10
```

## Common Causes

1. **ECMP without flow tracking** — different paths selected for forward and return
2. **Dual-ISP with asymmetric BGP** — inbound and outbound traffic use different ISP links
3. **Policy routing** — source-based routing sends return traffic via a different interface
4. **Misconfigured static routes** — return route points to a different next-hop

## Fixing Asymmetric Routing with Policy Routing

When a server has two upstream links, use policy routing to ensure replies leave through the same interface:

```bash
# Create two routing tables (one per ISP)
echo "100 isp1" >> /etc/iproute2/rt_tables
echo "200 isp2" >> /etc/iproute2/rt_tables

# Table 100: default route via ISP1 gateway
ip route add default via 203.0.113.1 table 100

# Table 200: default route via ISP2 gateway
ip route add default via 198.51.100.1 table 200

# Rule: if source IP is on ISP1's subnet, use table 100
ip rule add from 203.0.113.5 table 100
# Rule: if source IP is on ISP2's subnet, use table 200
ip rule add from 198.51.100.5 table 200
```

## Handling Asymmetric Routing with Stateful Firewalls

If you cannot avoid asymmetric routing (e.g., in a data center with BGP), configure iptables to accept related/established traffic regardless of direction:

```bash
# Disable strict reverse path filtering (rp_filter) to allow asymmetric flows
sysctl -w net.ipv4.conf.all.rp_filter=0
sysctl -w net.ipv4.conf.eth0.rp_filter=0

# For connection tracking with asymmetric routing, use loose mode
# net.ipv4.conf.all.rp_filter=2 means loose mode
sysctl -w net.ipv4.conf.all.rp_filter=2
```

## Verifying the Fix

```bash
# After applying policy routing, confirm symmetry
ip route get 10.20.0.10 from 203.0.113.5
# Response should show the correct source interface

# Test with TCP to confirm stateful connections work
curl -v http://10.20.0.10

# Check conntrack for confirmed bidirectional entries
conntrack -L | grep "10.20.0.10"
```

## Conclusion

Asymmetric routing is subtle and often manifests as intermittent connection drops rather than complete failures. The fix depends on the cause: policy routing for multi-homed servers, rp_filter relaxation for transit scenarios, and proper BGP route policies for data center environments. Always verify both forward and return paths when diagnosing mysterious TCP failures.
