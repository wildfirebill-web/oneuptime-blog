# How to Handle DHCP Pool Exhaustion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Pool Exhaustion, Networking, Troubleshooting, Sysadmin

Description: DHCP pool exhaustion occurs when all addresses in a scope are leased and new clients cannot obtain an IP, resolved by reducing lease times, expanding the pool, reclaiming stale leases, or...

## Detecting Pool Exhaustion

```bash
# Count active leases

ACTIVE=$(grep -c "binding state active" /var/lib/dhcp/dhcpd.leases)
POOL_SIZE=100  # Your pool size

echo "Active leases: $ACTIVE / $POOL_SIZE"
if [ "$ACTIVE" -ge "$POOL_SIZE" ]; then
    echo "WARNING: Pool is exhausted!"
fi

# Windows Server
# Get-DhcpServerv4ScopeStatistics -ScopeId 192.168.1.0
```

Server logs will show:
```text
DHCPDISCOVER from aa:bb:cc:dd:ee:ff via eth0: no free leases
```

## Solution 1: Reduce Lease Time

Shorter leases return addresses to the pool faster:

```text
# /etc/dhcp/dhcpd.conf
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.50 192.168.1.200;
    option routers 192.168.1.1;
    # Reduce from 24h to 4h to reclaim addresses faster
    default-lease-time 14400;    # 4 hours
    max-lease-time 28800;        # 8 hours max
}
```

## Solution 2: Expand the Address Pool

```text
# Original scope: .50 to .200 (150 addresses)
# Expanded: .50 to .240 (190 addresses)

subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.50 192.168.1.240;   # Expanded pool
    option routers 192.168.1.1;
}
```

## Solution 3: Reclaim Stale Leases

Manually expire leases for devices that are no longer on the network:

```bash
# Find leases that haven't been renewed in > 7 days
python3 << 'EOF'
import re
from datetime import datetime, timezone

with open("/var/lib/dhcp/dhcpd.leases") as f:
    content = f.read()

# Parse each lease block
lease_blocks = re.findall(r'lease ([\d.]+) \{(.*?)\}', content, re.DOTALL)
now = datetime.now(timezone.utc)
stale = []

for ip, block in lease_blocks:
    ends_match = re.search(r'ends \d+ (\d+/\d+/\d+ \d+:\d+:\d+)', block)
    state_match = re.search(r'binding state (\w+)', block)
    if ends_match and state_match and state_match.group(1) == 'active':
        try:
            ends = datetime.strptime(ends_match.group(1), "%Y/%m/%d %H:%M:%S")
            ends = ends.replace(tzinfo=timezone.utc)
            if ends < now:
                stale.append(ip)
        except ValueError:
            pass

print(f"Stale expired leases: {stale}")
EOF
```

## Solution 4: Subnet the Network

If the single subnet is too small, divide clients into multiple smaller subnets:

```python
import ipaddress

# Original: 192.168.1.0/24 (254 hosts)
# Split into two /25s for different departments
original = ipaddress.IPv4Network("192.168.1.0/24")
for subnet in original.subnets(new_prefix=25):
    print(f"  {subnet}  ({subnet.num_addresses - 2} hosts)")
```

## Solution 5: Use /23 Instead of /24

```text
# Extend to a /23 block to double address space
subnet 192.168.0.0 netmask 255.255.254.0 {
    range 192.168.0.50 192.168.1.250;   # ~450 addresses
    option routers 192.168.0.1;
}
```

## Monitoring and Alerting

```bash
# Alert when pool > 80% full
#!/bin/bash
LEASES=$(grep -c "binding state active" /var/lib/dhcp/dhcpd.leases 2>/dev/null || echo 0)
POOL=150
THRESHOLD=80

PERCENT=$(( LEASES * 100 / POOL ))
if [ "$PERCENT" -gt "$THRESHOLD" ]; then
    echo "DHCP pool ${PERCENT}% full (${LEASES}/${POOL})" | \
        logger -p daemon.warning -t dhcp-monitor
fi
```

## Key Takeaways

- First response to exhaustion: reduce lease time to reclaim addresses faster.
- Second response: expand the pool range or use a larger subnet.
- Proactively monitor pool utilization and alert at 80% to get ahead of exhaustion.
- Segmenting large flat networks into VLANs distributes the load across multiple smaller scopes.
