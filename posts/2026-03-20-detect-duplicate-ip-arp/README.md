# How to Detect Duplicate IP Addresses Using ARP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, ARP, IPv4, Troubleshooting

Description: Learn how to use ARP techniques including arping, Wireshark, and Python scripts to detect and resolve duplicate IP addresses.

## Why Duplicate IPs Occur

Duplicate IP addresses happen when two hosts on the same subnet share the same IP address. Common causes:

- Manual static IP assignment without checking availability
- DHCP pool overlap with static assignments
- Network device migration without updating DNS/DHCP records
- Virtual machine cloning without changing IP

## Symptoms of Duplicate IPs

- Intermittent connectivity (two hosts fight over the IP)
- "Duplicate IP address" warnings in system logs
- ARP table entries flapping between two MACs
- Applications connecting to the wrong host

## Method 1: arping Duplicate Detection Mode

```bash
# -D flag exits with code 1 if a reply is received
arping -D -I eth0 -c 2 192.168.1.50

# Check result
if [ $? -eq 0 ]; then
    echo "No duplicate: 192.168.1.50 is available"
else
    echo "DUPLICATE DETECTED: 192.168.1.50 is already in use"
fi
```

## Method 2: Python Script with Scapy

```python
from scapy.all import ARP, Ether, srp

def detect_duplicate_ip(target_ip, iface='eth0'):
    """Send gratuitous ARP and collect all replies to find duplicates."""
    pkt = Ether(dst='ff:ff:ff:ff:ff:ff') / ARP(
        op=1,
        psrc=target_ip,   # Sender IP = target (gratuitous)
        pdst=target_ip    # Target IP = same
    )
    results, _ = srp(pkt, timeout=2, iface=iface, verbose=False)
    
    macs = [rcv[ARP].hwsrc for _, rcv in results]
    
    if len(macs) == 0:
        print(f"{target_ip}: No host found")
    elif len(macs) == 1:
        print(f"{target_ip}: Owned by {macs[0]}")
    else:
        print(f"DUPLICATE IP DETECTED: {target_ip}")
        for mac in macs:
            print(f"  Claimed by MAC: {mac}")

detect_duplicate_ip('192.168.1.50')
```

## Method 3: Monitor ARP Table for Flapping

```bash
#!/bin/bash
# Detect MAC flapping in ARP table (sign of duplicate IPs)
PREV=""
while true; do
    CURR=$(ip neigh show | awk '{print $1, $5}' | sort)
    if [ -n "$PREV" ] && [ "$CURR" != "$PREV" ]; then
        echo "$(date): ARP table changed!"
        diff <(echo "$PREV") <(echo "$CURR")
    fi
    PREV="$CURR"
    sleep 5
done
```

## Method 4: Wireshark Detection

In Wireshark, apply the filter:

```
arp.duplicate-address-detected
```

Wireshark automatically flags ARP packets where the same IP claims different MAC addresses across multiple packets.

## Method 5: Scan Subnet for Duplicate IP Holders

```python
from scapy.all import ARP, Ether, srp
import ipaddress
from collections import defaultdict

def scan_for_duplicates(subnet='192.168.1.0/24', iface='eth0'):
    """ARP scan entire subnet and report duplicate IPs."""
    network = ipaddress.ip_network(subnet)
    
    # Broadcast ARP request to entire subnet
    pkt = Ether(dst='ff:ff:ff:ff:ff:ff') / ARP(pdst=str(subnet))
    results, _ = srp(pkt, timeout=3, iface=iface, verbose=False)
    
    ip_mac = defaultdict(list)
    for _, rcv in results:
        ip_mac[rcv[ARP].psrc].append(rcv[ARP].hwsrc)
    
    for ip, macs in ip_mac.items():
        if len(macs) > 1:
            print(f"DUPLICATE: {ip} claimed by: {', '.join(set(macs))}")
        else:
            print(f"OK: {ip} → {macs[0]}")

scan_for_duplicates('192.168.1.0/24')
```

## Resolving Duplicate IP Addresses

1. Identify both hosts claiming the IP (using the methods above)
2. Determine which host should own the IP
3. On the conflicting host: change IP or enable DHCP
4. Clear ARP cache on affected hosts: `ip neigh flush all`
5. Add static DHCP reservations to prevent recurrence

## Key Takeaways

- `arping -D` is the simplest tool for checking if an IP is already in use.
- Scapy can detect multiple hosts claiming the same IP.
- ARP table flapping (same IP, changing MACs) is a strong indicator of duplicate IPs.
- Wireshark's `arp.duplicate-address-detected` filter automates detection.

**Related Reading:**

- [How to Use arping to Test ARP Resolution](https://oneuptime.com/blog/post/2026-03-20-arping-test-arp-resolution/view)
- [How to Understand Gratuitous ARP and Its Uses](https://oneuptime.com/blog/post/2026-03-20-gratuitous-arp-uses/view)
- [How to Configure DHCP Reservations for Static Assignments](https://oneuptime.com/blog/post/2026-03-20-dhcp-reservations-static-ip/view)
