# How to Manage IPv4 Address Exhaustion in a Growing Enterprise

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Address Exhaustion, Network Design, IPv6, NAT, Optimization

Description: Learn strategies to extend the life of your IPv4 private address space when your network is running out of IP addresses, including reclamation, right-sizing, and IPv6 adoption.

## Signs of IPv4 Exhaustion

- DHCP pools running out of addresses (devices can't connect)
- IPAM showing 90%+ utilization across major blocks
- New projects waiting for IP allocations
- Multiple overlapping NAT layers (double-NAT)

## Step 1: Audit Current Usage

```python
# Scan all subnets and measure actual utilization

import subprocess
from ipaddress import ip_network

def measure_utilization(subnet_cidr):
    net = ip_network(subnet_cidr)
    active = 0

    for host in net.hosts():
        result = subprocess.run(
            ['ping', '-c', '1', '-W', '1', str(host)],
            capture_output=True
        )
        if result.returncode == 0:
            active += 1

    utilization = active / net.num_addresses * 100
    return {'subnet': subnet_cidr, 'active': active,
            'total': net.num_addresses, 'utilization': f"{utilization:.1f}%"}

# Identify wasted space
subnets = ['10.1.0.0/22', '10.1.4.0/22', '10.1.8.0/22']
for subnet in subnets:
    info = measure_utilization(subnet)
    print(f"{info['subnet']}: {info['active']}/{info['total']} ({info['utilization']})")
```

## Step 2: Right-Size Over-Allocated Subnets

Many networks over-allocate out of caution, wasting 70-80% of space:

```text
Before (over-allocated):
  Department A: 10.1.0.0/22 (1022 usable, 45 actual hosts)
  Department B: 10.1.4.0/22 (1022 usable, 12 actual hosts)
  Department C: 10.1.8.0/22 (1022 usable, 8 actual hosts)
  Total wasted: ~2900 addresses

After (right-sized):
  Department A: 10.1.0.0/26 (62 usable, 45 hosts - 20% headroom)
  Department B: 10.1.0.64/27 (30 usable, 12 hosts - 150% headroom)
  Department C: 10.1.0.96/27 (30 usable, 8 hosts - 275% headroom)
  Reclaimed: ~2850 addresses from just 3 subnets
```

## Step 3: Consolidate Fragmented Allocations

```python
from ipaddress import ip_network, collapse_addresses

# Fragmented allocations that can be consolidated
fragmented = [
    '10.5.1.0/24', '10.5.2.0/24',    # Could be one /23
    '10.5.4.0/24', '10.5.5.0/24',    # Could be one /23
    '10.5.6.0/24', '10.5.7.0/24',    # Could be one /23
]

# Find summary routes
nets = [ip_network(n) for n in fragmented]
summaries = list(collapse_addresses(nets))
print(f"Original prefixes: {len(fragmented)}")
print(f"After summarization: {len(summaries)} prefixes: {summaries}")
```

## Step 4: Reclaim Unused Private Space

Instead of 10.0.0.0/8, consider 172.16.0.0/12 for small networks:

```text
Available private space review:
  10.0.0.0/8 - Currently using 10.0.0.0-10.10.255.255 (11 /16s)
  10.11.0.0/8 through 10.255.0.0/16 - completely unused!

  Action: Audit and document the unused range, make it available for
  new projects instead of requesting public IP space.
```

## Step 5: Implement Strict DHCP Lease Management

Short lease times for transient clients free up addresses faster:

```bash
# Guest network: 30 minute leases (free up IPs quickly)
# Office network: 24 hour leases (stable for working day)
# Server network: Static/reserved (never changes)

# Monitor pool utilization
cat /var/lib/misc/dnsmasq.leases | wc -l
# If approaching pool size, either:
# 1. Expand the pool
# 2. Reduce lease times
# 3. Implement client-specific shorter leases
```

## Step 6: Use IPv6 for New Workloads

The long-term solution: adopt IPv6 for new deployments while maintaining IPv4 for existing:

```bash
# Enable IPv6 on a Linux server (dual-stack)
# /etc/network/interfaces
iface eth0 inet6 static
    address 2001:db8:1:1::10/64
    gateway 2001:db8:1:1::1

# Configure applications to prefer IPv6
# /etc/gai.conf
precedence ::1/128       50
precedence ::/0          40
precedence 2002::/16     30
precedence ::/96         20
precedence ::ffff:0:0/96 10   # Deprioritize IPv4-mapped IPv6

# For new microservices/containers, deploy IPv6-only internally
# with 64NAT (NAT64) for IPv4 internet access
```

## Step 7: Implement Larger NAT Pools

If you must stay IPv4-only, use Carrier-Grade NAT (CGN):

```text
RFC 6598 shared address space: 100.64.0.0/10
  Used between ISP and customer, or large internal NAT pools
  4 million addresses: 100.64.0.0 - 100.127.255.255

This allows double-NAT without conflicting with typical RFC 1918 space.
```

## Conclusion

IPv4 exhaustion is managed through a combination of: auditing actual utilization (typically 20-40% waste), right-sizing over-allocated subnets, reclaiming unused ranges, and implementing short DHCP leases for transient clients. These measures can extend private address space life by years. The long-term solution is IPv6 adoption for new workloads, reducing dependence on ever-larger RFC 1918 allocations.
