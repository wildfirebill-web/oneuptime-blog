# How to Troubleshoot Routing Loops in IPv4 Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, IPv4, Routing, Troubleshooting, BGP, OSPF

Description: Learn how to identify, diagnose, and fix routing loops in IPv4 networks using traceroute, TTL analysis, and routing protocol tools.

## Introduction

A routing loop occurs when packets continuously circulate between two or more routers without ever reaching their destination. Each router forwards the packet to the next, believing the destination is reachable via that path — creating an infinite cycle. The primary protection against endless loops is the IP TTL (Time to Live) field, which decrements at each hop and eventually triggers an ICMP Time Exceeded message.

## Identifying a Routing Loop

The most direct symptom is traceroute showing repeated hops or asterisks.

Run traceroute to identify the loop:

```bash
# Linux traceroute
traceroute 10.10.5.1

# Output showing a loop (same IPs repeat):
# 1  192.168.1.1  1.2 ms
# 2  10.0.0.1     2.5 ms
# 3  10.0.1.1     3.1 ms
# 4  10.0.0.1     2.8 ms   <-- loop starts here
# 5  10.0.1.1     3.2 ms
# ...
```

You can also check MTR for a real-time view:

```bash
# MTR shows packet loss and repeated hops
mtr --report --report-cycles 10 10.10.5.1
```

## Common Causes

1. **Misconfigured static routes** — two routers each have a static route pointing to the other for the same subnet
2. **Redistribution loops** — routes redistributed between OSPF and BGP without proper filtering
3. **Default route propagation** — all routers forward unknown traffic to a default route that loops back

## Diagnosing with Routing Tables

Check the routing table on each suspected router:

```bash
# Linux: show the routing table
ip route show

# Show the specific route that may be looping
ip route get 10.10.5.1

# On Cisco IOS (if accessible via console or SSH)
# show ip route 10.10.5.1
# show ip route
```

## Fixing Static Route Loops

If two routers (Router A: 192.168.1.1, Router B: 192.168.2.1) are looping:

```bash
# On Router A (Linux): remove the bad static route and add correct one
ip route del 10.10.5.0/24 via 192.168.2.1
ip route add 10.10.5.0/24 via 10.0.0.254   # correct next-hop

# Persist the fix in /etc/network/interfaces or netplan
```

## Preventing Loops in Dynamic Routing

For OSPF, verify adjacency and route origins:

```bash
# Check OSPF neighbors (FRR/Quagga on Linux)
vtysh -c "show ip ospf neighbor"

# Check OSPF route database for duplicate entries
vtysh -c "show ip ospf database"
```

For BGP, use AS path filtering to prevent re-advertisement:

```bash
# Verify BGP routes aren't looping via AS path
vtysh -c "show ip bgp 10.10.5.0/24"
# If your own AS appears in the AS_PATH, the route is looping back
```

## Using TTL to Confirm a Loop

```bash
# Send a packet with a low TTL to see where it loops
ping -t 5 10.10.5.1    # macOS: -m 5
ping -c 1 -t 5 10.10.5.1   # Linux

# The ICMP Time Exceeded response reveals the looping router's IP
```

## Conclusion

Routing loops are disruptive but diagnosable. Start with traceroute to spot the repeating pattern, then investigate the routing tables on the involved routers. Whether caused by misconfigured static routes or redistribution errors, the fix almost always involves correcting the next-hop or adding proper route filtering. Monitoring with tools like OneUptime can alert you when TTL-exceeded errors spike — an early warning sign of loop activity.
