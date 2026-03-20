# How to Identify and Reclaim Unused IPv4 Address Space

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Address Management, IPAM, Network Optimization, Reclamation

Description: Learn how to identify unused or underutilized IPv4 address blocks within your network and reclaim them to extend the life of your private address space.

## Why Reclaim IPv4 Addresses?

Enterprise networks often allocate large blocks for anticipated growth that never happens, leaving vast swaths of IP space sitting idle. Reclaiming these addresses:
- Extends the life of your private address space
- Improves IPAM accuracy
- Reduces routing table bloat
- Provides address space for new projects or cloud connectivity

## Step 1: Scan for Active Hosts

```bash
# Scan a subnet for active hosts

nmap -sn 192.168.1.0/24 -oG - | awk '/Up$/{print $2}' | sort -t. -k4 -n

# Scan an entire block for active hosts (may take a while for large ranges)
nmap -sn 10.0.0.0/16 -oG - | awk '/Up$/{print $2}' > /tmp/active-hosts.txt

# Count active hosts
wc -l /tmp/active-hosts.txt
```

## Step 2: Identify Unused Subnets

```python
from ipaddress import ip_network, ip_address
import subprocess
import re

def scan_subnet_utilization(subnets):
    """Check which subnets have active hosts using ping sweep."""
    results = {}

    for subnet_cidr in subnets:
        net = ip_network(subnet_cidr)
        active_count = 0

        # Quick ping sweep (adjust based on network size)
        for host in list(net.hosts())[:20]:  # Sample first 20 hosts
            result = subprocess.run(
                ['ping', '-c', '1', '-W', '1', str(host)],
                capture_output=True
            )
            if result.returncode == 0:
                active_count += 1

        utilization = (active_count / net.num_addresses) * 100
        results[subnet_cidr] = {
            'active_sampled': active_count,
            'total_hosts': net.num_addresses - 2,
            'utilization_estimate': utilization,
            'status': 'ACTIVE' if active_count > 0 else 'UNUSED',
        }

    return results

# Check subnet utilization
subnets_to_check = [
    '10.1.0.0/24', '10.1.1.0/24', '10.1.2.0/24',
    '10.1.3.0/24', '10.1.4.0/24', '10.1.5.0/24',
]

results = scan_subnet_utilization(subnets_to_check)
for subnet, data in results.items():
    print(f"{subnet}: {data['status']} (sampled {data['active_sampled']} active hosts)")
```

## Step 3: Check DHCP Lease Database

```bash
# ISC DHCPD - find subnets with no active leases
dhcpd_leases='/var/lib/dhcp/dhcpd.leases'

# Extract active leases
grep "^lease" $dhcpd_leases | awk '{print $2}' | sort > /tmp/dhcp-leased.txt

# Find which subnets are using leases
python3 << 'EOF'
from ipaddress import ip_address, ip_network

# Subnets to check
subnets = [ip_network('10.1.0.0/24'), ip_network('10.1.1.0/24')]

# Load DHCP leases
with open('/tmp/dhcp-leased.txt') as f:
    leased_ips = set(line.strip() for line in f if line.strip())

for subnet in subnets:
    active_leases = [ip for ip in leased_ips
                     if ip_address(ip) in subnet]
    print(f"{subnet}: {len(active_leases)} active DHCP leases")
EOF
```

## Step 4: Check ARP Tables for Activity

```bash
# Gather ARP entries from routers (indicates active hosts)
# On Cisco IOS:
show arp | grep "GigabitEthernet0/0" | awk '{print $2}' > /tmp/arp-hosts.txt

# On Linux:
arp -n | awk '{print $1}' | grep -v "^Address" > /tmp/arp-hosts.txt

# Check when ARP entries were last seen
# cat /proc/net/arp shows the current ARP cache
cat /proc/net/arp | grep "0x2" | awk '{print $1}'
# 0x2 = ARP entry for a host (complete)
```

## Step 5: Identify Over-Allocated Subnets

```python
from ipaddress import ip_network

def check_subnet_sizing(subnet_cidr, actual_hosts):
    """Check if a subnet is over-allocated for its actual usage."""
    net = ip_network(subnet_cidr)
    usable = net.num_addresses - 2
    utilization = (actual_hosts / usable) * 100

    # Recommend right-sizing
    optimal_prefix = 32
    for prefix in range(30, 21, -1):
        hosts_needed = actual_hosts * 2  # 2x actual for growth
        if 2 ** (32 - prefix) - 2 >= hosts_needed:
            optimal_prefix = prefix

    print(f"Subnet {subnet_cidr}: {actual_hosts}/{usable} hosts ({utilization:.1f}%)")
    if optimal_prefix < net.prefixlen:
        print(f"  Over-allocated! Optimal size: /{optimal_prefix} ({2**(32-optimal_prefix)-2} hosts)")
        # Calculate reclaimable space
        waste = net.num_addresses - 2 ** (32 - optimal_prefix)
        print(f"  Reclaimable: {waste} addresses")

# Example: /22 with only 15 hosts
check_subnet_sizing('10.1.0.0/22', 15)
# Suggests right-sizing to /27 (30 usable hosts)
```

## Step 6: Reclaim and Reallocate

After identifying unused space:

```bash
# Document what you're reclaiming (before making changes)
echo "Reclaiming: 10.1.5.0/24 - no active hosts found" >> /var/log/ipam-changes.log

# Update DHCP server to stop serving the reclaimed subnet
# Remove from dhcpd.conf and reload

# Remove routes to the reclaimed subnet from routers
# On Cisco IOS:
no ip route 10.1.5.0 255.255.255.0

# Update IPAM (NetBox) to mark as available
curl -s -X PATCH http://netbox/api/ipam/prefixes/<id>/ \
  -H "Authorization: Token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "available", "description": "Reclaimed 2026-03-20"}'
```

## Conclusion

IPv4 space reclamation starts with scanning for active hosts (nmap ping sweep), checking DHCP lease databases, and reviewing ARP tables. Identify subnets with zero or very few active hosts, right-size over-allocated blocks, and document changes in IPAM. Update DHCP, routing, and IPAM to reflect reclaimed space. Regular quarterly audits using these techniques can recover 20-40% of allocated-but-unused address space in typical enterprise networks.
