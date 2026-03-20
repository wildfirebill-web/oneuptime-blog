# How to Integrate IPAM with DHCPv6 for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPAM, DHCPv6, Kea, NetBox, Automation

Description: Integrate IPAM tools with DHCPv6 servers to synchronize address allocations, import lease data, and generate DHCPv6 host reservations from IPAM records.

## Introduction

IPAM-DHCPv6 integration creates bidirectional synchronization: IPAM feeds host reservations to the DHCPv6 server, and the DHCPv6 server feeds lease data back to IPAM. This ensures the IPAM record reflects actual addresses in use and eliminates the manual step of adding DHCPv6 reservations separately from IPAM entries.

## Architecture

```mermaid
flowchart LR
    NETBOX["NetBox\n(IPAM)"] -->|"Generate reservations"| KEA["Kea DHCPv6\nServer"]
    KEA -->|"Lease events via webhooks"| NETBOX
    KEA -->|"Lease file"| SYNC["Sync Script"]
    SYNC -->|"Update IPAM records"| NETBOX
```

## Step 1: Generate Kea Reservations from NetBox

```python
#!/usr/bin/env python3
# netbox_to_kea_reservations.py
# Generate Kea DHCPv6 host reservations from NetBox IPAM records

import pynetbox
import json

nb = pynetbox.api("http://netbox.internal", token="your-token")

def generate_kea_reservations(subnet: str) -> list:
    """Generate Kea host reservation list from NetBox IP addresses."""
    reservations = []

    ip_addresses = nb.ipam.ip_addresses.filter(parent=subnet, status="active")
    for ip in ip_addresses:
        # Only create reservation if MAC/DUID is known
        if not ip.assigned_object:
            continue

        addr = str(ip.address).split('/')[0]
        description = ip.description or str(ip.dns_name) or ""

        # Get MAC from assigned interface
        iface = ip.assigned_object
        mac = str(getattr(iface, 'mac_address', '')) if iface else ''

        if mac:
            reservation = {
                "hw-address": mac,
                "ip-addresses": [addr],
                "hostname": str(ip.dns_name).rstrip('.') if ip.dns_name else "",
                "comment": description
            }
            reservations.append(reservation)

    return reservations

# Generate reservations for a specific subnet
subnet = "2001:db8:0001:0001::/64"
reservations = generate_kea_reservations(subnet)

# Output Kea configuration snippet
kea_config_snippet = {
    "Dhcp6": {
        "subnet6": [{
            "subnet": subnet,
            "reservations": reservations
        }]
    }
}

print(json.dumps(kea_config_snippet, indent=2))
```

## Step 2: Import Kea Leases into NetBox

```python
#!/usr/bin/env python3
# kea_leases_to_netbox.py
# Import active Kea DHCPv6 leases into NetBox

import json
import urllib.request
import pynetbox
import ipaddress

nb = pynetbox.api("http://netbox.internal", token="your-token")

def get_kea_leases() -> list:
    """Get all active DHCPv6 leases from Kea REST API."""
    payload = json.dumps({
        "command": "lease6-get-all",
        "service": ["dhcp6"]
    }).encode()

    req = urllib.request.Request(
        "http://[::1]:8000",
        data=payload,
        headers={"Content-Type": "application/json"}
    )

    with urllib.request.urlopen(req, timeout=10) as resp:
        result = json.load(resp)
        return result.get("arguments", {}).get("leases", [])

def sync_leases_to_netbox(leases: list):
    """Create or update NetBox IP address records for active leases."""
    for lease in leases:
        if lease.get("type") != "IA_NA":  # Only /128 addresses, not PD
            continue

        ip_addr = lease.get("ip-address")
        duid = lease.get("duid", "")
        hostname = lease.get("hostname", "")

        if not ip_addr:
            continue

        # Normalize address
        normalized = str(ipaddress.ip_address(ip_addr)) + "/128"

        # Check if IP already exists in NetBox
        existing = nb.ipam.ip_addresses.filter(address=normalized)

        if existing:
            # Update existing record
            ip_obj = list(existing)[0]
            nb.ipam.ip_addresses.update([{
                "id": ip_obj.id,
                "description": f"DHCPv6 lease | DUID: {duid[:20]}",
                "dns_name": hostname,
                "status": "active"
            }])
        else:
            # Create new record for lease
            nb.ipam.ip_addresses.create({
                "address": normalized,
                "description": f"DHCPv6 lease | DUID: {duid[:20]}",
                "dns_name": hostname,
                "status": "active",
                "tags": [{"slug": "dhcpv6-lease"}]
            })
            print(f"Created IPAM record: {normalized} ({hostname})")

leases = get_kea_leases()
print(f"Processing {len(leases)} active leases...")
sync_leases_to_netbox(leases)
print("Sync complete")
```

## Step 3: Webhook Sync (Real-time)

Configure Kea to call a webhook when leases change:

```json
// Kea configuration: lease4/6 callouts
// /etc/kea/kea-dhcp6.conf (snippet)
{
  "Dhcp6": {
    "hooks-libraries": [{
      "library": "/usr/lib/kea/hooks/libdhcp_run_script.so",
      "parameters": {
        "name": "/etc/kea/hooks/netbox_sync.sh",
        "sync": false
      }
    }]
  }
}
```

```bash
#!/bin/bash
# /etc/kea/hooks/netbox_sync.sh
# Called by Kea on lease events

NETBOX_URL="http://netbox.internal"
NETBOX_TOKEN="${NETBOX_TOKEN}"

case "$KEA_LEASE6_ACTION" in
    "assign"|"renew")
        # Create/update IPAM record
        curl -s -X POST "${NETBOX_URL}/api/ipam/ip-addresses/" \
            -H "Authorization: Token ${NETBOX_TOKEN}" \
            -H "Content-Type: application/json" \
            -d "{
                \"address\": \"${KEA_LEASE6_ADDRESS}/128\",
                \"description\": \"DHCPv6 ${KEA_LEASE6_DUID}\",
                \"status\": \"active\"
            }" > /dev/null
        ;;
    "expire"|"release")
        # Mark as deprecated
        # (implementation depends on finding the IP ID first)
        ;;
esac
```

## Step 4: Run Periodic Reconciliation

```bash
# Crontab entry for periodic IPAM-DHCPv6 sync
*/30 * * * * /usr/local/bin/kea_leases_to_netbox.py >> /var/log/ipam-sync.log 2>&1
0 2 * * * /usr/local/bin/netbox_to_kea_reservations.py | kea-admin lease-reload dhcp6 >> /var/log/ipam-sync.log 2>&1
```

## Conclusion

IPAM-DHCPv6 integration requires two synchronization directions: IPAM to DHCPv6 for host reservations (ensuring specific devices get specific addresses), and DHCPv6 to IPAM for lease records (ensuring IPAM reflects dynamically assigned addresses). Use Kea's run_script hook for real-time sync on lease events, supplemented by periodic batch reconciliation to catch any missed events. The `lease6-get-all` Kea REST API command provides a complete current lease snapshot for batch import.
