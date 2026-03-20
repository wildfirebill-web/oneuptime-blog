# How to Troubleshoot IPv6 in Data Center Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Troubleshooting, Data Center, NDP, BGP, Networking

Description: A systematic guide to troubleshooting IPv6 connectivity, routing, and NDP issues in data center environments.

## Troubleshooting Methodology

Follow a layered approach: physical → link-local → NDP → routing → application.

## Step 1: Verify Link-Local Connectivity

Link-local addresses are auto-configured. If they are missing, there is a physical or driver issue:

```bash
# Check IPv6 addresses on all interfaces

ip -6 addr show

# Expected output for a healthy interface:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet6 fe80::1234:5678:abcd:ef01/64 scope link

# Ping the router's link-local address
ping6 fe80::1%eth0
```

## Step 2: Check NDP Neighbor Table

If link-local is up but global addresses are unreachable, check the NDP table:

```bash
# View NDP neighbor cache
ip -6 neigh show

# Expected state: REACHABLE or STALE (not FAILED or missing)
# 2001:db8:1::1 dev eth0 lladdr 00:1a:2b:3c:4d:5e REACHABLE

# Manually trigger NDP resolution
ping6 2001:db8:1::1
ip -6 neigh show 2001:db8:1::1
```

## Step 3: Verify Router Advertisements

Hosts receive their default gateway and prefix info via RA. Check if RAs are arriving:

```bash
# Use radvdump to capture RA messages
apt install radvd
radvdump

# Or use tcpdump to capture ICMPv6 type 134 (Router Advertisement)
tcpdump -i eth0 'icmp6 and ip6[40] == 134'
```

## Step 4: Check IPv6 Routing Table

Verify the routing table has the correct entries:

```bash
# Show IPv6 routes
ip -6 route show

# Check for a default route (required for external connectivity)
# Expected: default via fe80::1 dev eth0 proto ra metric 100

# Test routing to a specific destination
ip -6 route get 2001:4860:4860::8888
```

## Step 5: Trace the Path

Use `traceroute6` or `mtr` to identify where packets are being dropped:

```bash
# Traceroute to Google's IPv6 DNS
traceroute6 2001:4860:4860::8888

# MTR for continuous path analysis
mtr -6 2001:4860:4860::8888
```

## Step 6: BGP Troubleshooting

If routes are not propagating, check BGP sessions:

```bash
# FRRouting BGP debugging
vtysh -c "show bgp ipv6 unicast summary"
vtysh -c "show bgp ipv6 unicast neighbors 2001:db8:1::2"
vtysh -c "show bgp ipv6 unicast 2001:db8:2::/48"
```

## Step 7: Firewall and ACL Checks

Drop counters on firewall rules can reveal blocked traffic:

```bash
# Check ip6tables counters
ip6tables -L -v -n --line-numbers | grep -v "0     0"

# Check nftables drop counters
nft list ruleset | grep -A2 "counter"
```

## Common Issues and Fixes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| No link-local address | Interface down or no IPv6 kernel support | `ip link set eth0 up`, check kernel |
| Missing default route | No RA received | Check router RA config, firewall on ICMPv6 |
| NDP entries FAILED | Firewall blocking NS/NA | Allow ICMPv6 types 135/136 |
| BGP session down | Wrong address family | Enable `address-family ipv6` |

## Conclusion

Systematic IPv6 troubleshooting in data centers proceeds from physical layer up through NDP, routing, and application. The most common issues involve ICMPv6 being blocked by firewalls (breaking NDP) or missing BGP address family configuration.
