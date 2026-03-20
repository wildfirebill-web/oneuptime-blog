# How to Debug Connectivity Issues Between Network Namespaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Network Namespaces, Debugging, Linux, IPv4, Connectivity, tcpdump, Ip route

Description: Learn how to systematically debug connectivity failures between Linux network namespaces using ip, tcpdump, iptables, and route inspection tools.

---

Connectivity failures between network namespaces are usually caused by missing routes, incorrect IP configuration, or iptables rules blocking traffic. This guide provides a systematic debugging approach.

## Checklist for Namespace Connectivity Debugging

```text
1. Interface is UP and has correct IP
2. Route exists to destination
3. ARP/neighbor table has the next hop
4. iptables/nftables not blocking traffic
5. IP forwarding enabled (if routing between namespaces)
6. veth pair both ends are UP
```

## Step 1: Check Interface State and IPs

```bash
# Check both sides of the connection

ip netns exec ns1 ip addr show
ip netns exec ns2 ip addr show

# Verify both veth ends are UP
ip netns exec ns1 ip link show
ip link show   # host side
```

## Step 2: Check Routing Tables

```bash
# Does ns1 know how to reach ns2?
ip netns exec ns1 ip route show

# Missing route: add it
ip netns exec ns1 ip route add 10.0.2.0/24 via 10.0.1.1
# Or default route
ip netns exec ns1 ip route add default via 10.0.1.1
```

## Step 3: Test with ping and Capture Traffic

```bash
# Ping from ns1 to ns2
ip netns exec ns1 ping -c3 10.0.2.2

# Capture in ns1 (sending side)
ip netns exec ns1 tcpdump -i veth1 -nn icmp &

# Capture in ns2 (receiving side)
ip netns exec ns2 tcpdump -i veth2 -nn icmp &

# Run ping
ip netns exec ns1 ping -c2 10.0.2.2

# If packets show up in ns1 but not ns2: veth or bridge issue
# If packets show up in ns2 but no reply: IP/route issue in ns2
```

## Step 4: Check iptables

```bash
# Check for DROP rules in namespace
ip netns exec ns2 iptables -L -n -v --line-numbers
ip netns exec ns2 iptables -t nat -L -n -v

# Check FORWARD chain in host namespace (if routing between namespaces)
iptables -L FORWARD -n -v
```

## Step 5: Check IP Forwarding

```bash
# Forwarding must be enabled to route between namespaces
cat /proc/sys/net/ipv4/ip_forward
# 0 = disabled, 1 = enabled

sysctl -w net.ipv4.ip_forward=1

# Namespace-specific forwarding
ip netns exec router-ns sysctl -w net.ipv4.ip_forward=1
```

## Step 6: Check ARP/Neighbor Tables

```bash
# Is the MAC address resolved?
ip netns exec ns1 ip neigh show

# If missing, trigger ARP
ip netns exec ns1 arping -I veth1 10.0.1.2

# Flush and retry
ip netns exec ns1 ip neigh flush all
ip netns exec ns1 ping -c1 10.0.1.2
```

## Common Fixes Summary

| Symptom | Cause | Fix |
|---------|-------|-----|
| No route to host | Missing route | `ip route add` |
| Packets sent but no reply | iptables DROP | Check FORWARD chain |
| Interface down | `ip link set up` not run | `ip link set veth up` |
| Ping works one way | Asymmetric routes | Check routes in both namespaces |

## Key Takeaways

- Always verify both ends of the veth pair are UP before debugging further.
- Use `tcpdump` in both source and destination namespaces to determine where packets are being lost.
- Check `iptables -L FORWARD` in the host; the FORWARD chain applies to packets routed between interfaces.
- Namespace-specific IP forwarding (`net.ipv4.ip_forward`) must be set in each namespace that routes traffic.
