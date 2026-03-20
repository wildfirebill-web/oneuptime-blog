# How to Manage IPv6 Address Conflicts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPAM, Address Conflicts, NDP, Network Troubleshooting

Description: Detect and resolve IPv6 address conflicts caused by duplicate address detection failures, SLAAC collisions, static assignment errors, and IPAM record divergence from actual usage.

## Introduction

IPv6 address conflicts are less common than IPv4 conflicts due to Duplicate Address Detection (DAD), but they still occur when DAD is bypassed, static addresses duplicate DHCPv6 assignments, or IPAM records diverge from actual deployment. This guide covers detection, prevention, and resolution.

## How IPv6 Conflicts Occur

| Scenario | Cause | Detection Method |
|----------|-------|-----------------|
| Static vs DHCPv6 overlap | Admin assigns static address already in DHCPv6 pool | IPAM check, DAD failure |
| SLAAC collision | Two interfaces generate same EUI-64 (rare) | DAD failure log |
| IPAM vs reality mismatch | IPAM shows address as available, but device still using it | NDP scan |
| Duplicate static assignment | Two admins assign same /128 | IPAM workflow, NDP |
| Privacy extension confusion | Old temporary address still active | NDP table check |

## Step 1: Detect Conflicts via DAD Failures

```bash
# Monitor kernel for DAD failures (Linux)

# DAD failure message appears when another device has the same address
dmesg | grep "IPv6 duplicate address"
# Output: IPv6: 2001:db8::1: IPv6 duplicate address detected!

# Real-time monitoring
journalctl -k -f | grep -i "duplicate address\|conflict\|dad"

# Detect DAD failures via syslog on the router
# On Cisco IOS:
# debug ipv6 nd
# Look for: IPv6-ND: DAD failed for <address> on <interface>
```

## Step 2: Scan NDP Tables for Conflicts

```python
#!/usr/bin/env python3
# detect_ipv6_conflicts.py
# Scan NDP tables and compare against IPAM

import subprocess
import ipaddress
import re
from collections import defaultdict

def get_ndp_table() -> dict:
    """Get the NDP neighbor table from all interfaces."""
    result = subprocess.run(
        ["ip", "-6", "neigh", "show"],
        capture_output=True, text=True
    )
    # Format: <addr> dev <iface> lladdr <mac> <state>
    NDP_RE = re.compile(
        r'(?P<addr>[0-9a-fA-F:]+) dev (?P<iface>\S+)'
        r'(?:\s+lladdr (?P<mac>[0-9a-fA-F:]+))?'
        r'\s+(?P<state>\S+)'
    )

    entries = defaultdict(list)
    for line in result.stdout.splitlines():
        m = NDP_RE.match(line)
        if m and m.group("state") not in ("FAILED", "NONE"):
            addr = str(ipaddress.ip_address(m.group("addr")))
            mac = m.group("mac") or "unknown"
            entries[addr].append({
                "interface": m.group("iface"),
                "mac": mac,
                "state": m.group("state")
            })

    return dict(entries)

def find_ndp_conflicts(ndp_table: dict) -> list:
    """Find addresses with multiple MACs (conflict indicator)."""
    conflicts = []
    for addr, entries in ndp_table.items():
        macs = {e["mac"] for e in entries if e["mac"] != "unknown"}
        if len(macs) > 1:
            conflicts.append({
                "address": addr,
                "macs": list(macs),
                "entries": entries
            })
    return conflicts

ndp = get_ndp_table()
conflicts = find_ndp_conflicts(ndp)

if conflicts:
    print(f"CONFLICTS DETECTED: {len(conflicts)}")
    for c in conflicts:
        print(f"  {c['address']} has multiple MACs: {', '.join(c['macs'])}")
else:
    print("No NDP conflicts detected")
```

## Step 3: Compare IPAM vs NDP Reality

```python
#!/usr/bin/env python3
# ipam_vs_ndp_reconcile.py
import pynetbox
import subprocess
import re
import ipaddress

nb = pynetbox.api("http://netbox.internal", token="your-token")

def get_ndp_active_addrs() -> set:
    """Get all currently active IPv6 addresses from NDP table."""
    result = subprocess.run(["ip", "-6", "neigh", "show"],
                            capture_output=True, text=True)
    addrs = set()
    for line in result.stdout.splitlines():
        parts = line.split()
        if parts and parts[-1] in ("REACHABLE", "STALE", "DELAY"):
            try:
                addrs.add(str(ipaddress.ip_address(parts[0])))
            except ValueError:
                pass
    return addrs

# Get IPAM "active" addresses in our /64
ipam_addresses = {
    str(ip.address).split('/')[0]
    for ip in nb.ipam.ip_addresses.filter(
        parent="2001:db8:0001:0001::/64",
        status="active"
    )
}

ndp_addresses = get_ndp_active_addrs()

# Addresses in IPAM but not seen via NDP (may be stale)
ipam_not_on_network = ipam_addresses - ndp_addresses
if ipam_not_on_network:
    print(f"In IPAM but not seen on network ({len(ipam_not_on_network)}):")
    for addr in sorted(ipam_not_on_network):
        print(f"  {addr}")

# Addresses on network but not in IPAM (untracked/shadow IT)
ndp_not_in_ipam = ndp_addresses - ipam_addresses
if ndp_not_in_ipam:
    print(f"\nOn network but not in IPAM ({len(ndp_not_in_ipam)}):")
    for addr in sorted(ndp_not_in_ipam):
        print(f"  {addr}  ← Untracked device!")
```

## Step 4: Prevent Conflicts with IPAM Workflows

```python
def assign_static_ipv6(address: str, hostname: str) -> bool:
    """
    Assign a static IPv6 address only if it's not already in use.
    Returns True if assignment succeeded.
    """
    # Check IPAM first
    existing = nb.ipam.ip_addresses.filter(address=address)
    if existing:
        print(f"CONFLICT: {address} already assigned to "
              f"{list(existing)[0].assigned_object}")
        return False

    # Check NDP table (live network)
    ndp = get_ndp_table()
    normalized = str(ipaddress.ip_address(address.split('/')[0]))
    if normalized in ndp:
        print(f"CONFLICT: {address} found in NDP table (active on network)")
        return False

    # Safe to assign
    nb.ipam.ip_addresses.create({
        "address": address,
        "description": hostname,
        "status": "active"
    })
    print(f"Assigned {address} to {hostname}")
    return True
```

## Conclusion

IPv6 address conflict management combines proactive prevention (IPAM-enforced allocation workflow, checking both IPAM records and live NDP table before static assignment) with reactive detection (DAD failure monitoring, NDP table scanning for duplicate MACs). The key operational practice is ensuring all IPv6 addresses - both static and DHCPv6 - flow through the IPAM system before deployment. Regular reconciliation between IPAM records and NDP table data identifies shadow IT devices and stale IPAM records that could cause future conflicts.
