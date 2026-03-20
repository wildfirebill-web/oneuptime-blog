# How to Troubleshoot Static Route Not Working on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Static Route, Troubleshooting, Linux, ip route, IPv4, Routing, Connectivity

Description: Learn how to systematically troubleshoot static routes that are not working on Linux, including route verification, gateway reachability, IP forwarding, and firewall checks.

---

A static route that doesn't work is usually caused by one of a few common issues: the route isn't in the table, the gateway is unreachable, IP forwarding is disabled, or a firewall is blocking traffic.

## Step 1: Verify the Route Exists

```bash
# Show all routes
ip route show

# Show routes to a specific network
ip route show 192.168.2.0/24

# If not present, add it
ip route add 192.168.2.0/24 via 10.0.0.1 dev eth0

# Check specific destination
ip route get 192.168.2.10
# 192.168.2.10 via 10.0.0.1 dev eth0 src 10.0.0.5
```

## Step 2: Verify Gateway Reachability

```bash
# The gateway must be directly reachable (on the same subnet)
ping 10.0.0.1

# Check ARP entry for gateway
ip neigh show 10.0.0.1
# 10.0.0.1 dev eth0 lladdr aa:bb:cc:dd:ee:01 REACHABLE

# If no ARP entry, gateway may be on a different subnet
# Check: is 10.0.0.1 in the same /prefix as the local interface?
ip addr show eth0 | grep "inet "
```

## Step 3: Check IP Forwarding (For Transit Routes)

```bash
# If this host is routing traffic for others, forwarding must be enabled
cat /proc/sys/net/ipv4/ip_forward

# 0 = disabled (packets not forwarded between interfaces)
# Enable:
sysctl -w net.ipv4.ip_forward=1
```

## Step 4: Use ip route get to Simulate Lookup

```bash
# Simulate how the kernel looks up a route
ip route get 192.168.2.10
# Should show the expected interface and gateway

# With specific source address
ip route get 192.168.2.10 from 10.0.0.5
# Useful for troubleshooting policy routing issues
```

## Step 5: Check for Firewall Blocking

```bash
# List iptables rules that might block forwarding
iptables -L FORWARD -n -v --line-numbers

# Check for DROP policies
iptables -L -n | grep "policy DROP"

# Test without firewall (careful in production!)
iptables -P FORWARD ACCEPT
```

## Step 6: Trace Packet Path with traceroute

```bash
# Trace route to destination
traceroute 192.168.2.10

# If traceroute hangs at the gateway:
#   - The gateway is not forwarding (check routing on that device)
#   - Return route is missing (asymmetric routing)
```

## Step 7: Check for Multiple Routing Tables

```bash
# Are there policy routing rules that override the main table?
ip rule show

# Check if traffic is being looked up in a different table
ip route show table 100
ip route show table all | grep 192.168.2.0
```

## Common Issues Summary

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Route not in table | Not added | `ip route add` |
| Route exists, no connectivity | Gateway unreachable | Check ARP, subnet mask |
| Forward traffic lost | IP forwarding off | `sysctl net.ipv4.ip_forward=1` |
| Traffic lost mid-path | Missing return route | Add reverse route |
| Intermittent | Route flapping or ECMP | Check routing daemon |

## Key Takeaways

- Always start with `ip route get <destination>` — it shows exactly which interface and gateway the kernel will use.
- A gateway must be in the directly connected subnet; if not, the kernel returns "no route to host".
- Enable `net.ipv4.ip_forward=1` on any host that needs to forward packets between interfaces.
- Use `ip rule show` to detect policy routing that may be overriding the main routing table.
