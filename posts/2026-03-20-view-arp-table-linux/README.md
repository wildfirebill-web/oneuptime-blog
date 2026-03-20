# How to View the ARP Table on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Linux, ARP, IPv4

Description: Learn how to view and interpret the ARP table on Linux using ip neigh, arp, and related commands.

## What Is the ARP Table?

The ARP (Address Resolution Protocol) table, also called the ARP cache, stores mappings between IPv4 addresses and MAC addresses. When your Linux host communicates with another device on the same subnet, it consults the ARP table to determine the destination MAC address.

## Viewing the ARP Table with `ip neigh`

The modern way to view ARP entries on Linux is the `ip neigh` command (part of `iproute2`):

```bash
ip neigh show
```

Sample output:

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
192.168.1.20 dev eth0 lladdr 00:11:22:33:44:55 STALE
192.168.1.50 dev eth0  FAILED
```

### Entry States

| State | Meaning |
|-------|---------|
| REACHABLE | Entry confirmed reachable recently |
| STALE | Not verified recently; will be probed on next use |
| DELAY | Waiting before probing |
| PROBE | Actively probing reachability |
| FAILED | Resolution failed, no response received |
| PERMANENT | Manually added static entry |
| NOARP | Interface does not use ARP |

## Filtering the ARP Table

```bash
# Show entries for a specific interface

ip neigh show dev eth0

# Show only reachable entries
ip neigh show nud reachable

# Show stale entries
ip neigh show nud stale

# Show entries for a specific IP
ip neigh show 192.168.1.1

# Show all states including failed
ip neigh show nud all
```

## Using the Legacy `arp` Command

The older `arp` command is still available on many systems (from `net-tools`):

```bash
# Show all ARP entries
arp -n

# Show with hostnames resolved
arp -a

# Show for a specific interface
arp -i eth0 -n
```

Sample output of `arp -n`:

```text
Address          HWtype  HWaddress           Flags Mask            Iface
192.168.1.1      ether   aa:bb:cc:dd:ee:ff   C                     eth0
192.168.1.20     ether   00:11:22:33:44:55   C                     eth0
```

## Parsing ARP Table in a Script

```bash
#!/bin/bash
# List all ARP-resolved neighbors with IP and MAC
ip neigh show | awk '/REACHABLE|STALE/ {print $1, $5}' | while read ip mac; do
    echo "IP: $ip  MAC: $mac"
done
```

## Python: Reading the ARP Cache

```python
import subprocess
import re

def get_arp_table():
    result = subprocess.run(['ip', 'neigh', 'show'], capture_output=True, text=True)
    entries = []
    for line in result.stdout.strip().split('\n'):
        parts = line.split()
        if len(parts) >= 5 and parts[4] != '<incomplete>':
            entries.append({'ip': parts[0], 'mac': parts[4], 'state': parts[-1]})
    return entries

for entry in get_arp_table():
    print(f"IP: {entry['ip']:18} MAC: {entry['mac']:20} State: {entry['state']}")
```

## Watching ARP Changes in Real Time

```bash
# Monitor ARP table changes
watch -n 1 'ip neigh show'

# Or use ip monitor
ip monitor neigh
```

## Key Takeaways

- Use `ip neigh show` for modern ARP table inspection on Linux.
- STALE entries are still valid but will be re-verified before use.
- FAILED entries indicate a device that did not respond to ARP requests.
- Use `ip neigh show nud reachable` to see only currently confirmed neighbors.

**Related Reading:**

- [How to Understand How ARP Maps IP Addresses to MAC Addresses](https://oneuptime.com/blog/post/2026-03-20-how-arp-maps-ip-to-mac-addresses/view)
- [How to Clear the ARP Cache](https://oneuptime.com/blog/post/2026-03-20-clear-arp-cache-linux/view)
- [How to Add Static ARP Entries](https://oneuptime.com/blog/post/2026-03-20-add-static-arp-entry-ip-neigh/view)
