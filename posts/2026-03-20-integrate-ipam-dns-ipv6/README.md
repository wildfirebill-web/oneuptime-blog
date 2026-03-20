# How to Integrate IPAM with DNS for IPv6 Records

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPAM, DNS, AAAA Records, Automation, NetBox

Description: Automate the creation and management of DNS AAAA and PTR records from IPAM address assignments, keeping DNS synchronized with the IPAM source of truth.

## Introduction

When IPv6 addresses are assigned in IPAM, corresponding DNS AAAA records and PTR records in the `ip6.arpa` zone should be created automatically. Manual DNS management creates drift between IPAM and DNS - automation ensures they stay synchronized.

## Generate AAAA Records from NetBox

```python
#!/usr/bin/env python3
# netbox_to_dns.py

# Generate DNS zone file additions from NetBox IPv6 addresses

import pynetbox
import ipaddress

nb = pynetbox.api("http://netbox.internal", token="your-token")

def ipv6_to_ptr_name(addr: str) -> str:
    """Convert IPv6 address to ip6.arpa PTR record name."""
    ip = ipaddress.ip_address(addr)
    full_hex = ip.exploded.replace(':', '')
    return '.'.join(reversed(full_hex)) + '.ip6.arpa.'

# Get all IPv6 addresses with DNS names
dns_records = []
for ip in nb.ipam.ip_addresses.filter(family=6, status="active"):
    if not ip.dns_name:
        continue

    addr = str(ip.address).split('/')[0]
    dns_name = str(ip.dns_name).rstrip('.')

    dns_records.append({
        "name": dns_name,
        "address": addr,
        "ptr": ipv6_to_ptr_name(addr)
    })

# Generate BIND zone additions
print("; AAAA Records")
print(f"; Generated: {__import__('datetime').datetime.now()}")
print()
for record in sorted(dns_records, key=lambda r: r['name']):
    # AAAA record
    hostname = record['name'].split('.')[0]
    print(f"{hostname:<25} IN  AAAA  {record['address']}")

print()
print("; PTR Records (ip6.arpa)")
for record in sorted(dns_records, key=lambda r: r['name']):
    print(f"{record['ptr']} IN  PTR  {record['name']}.")
```

## Sync IPAM to BIND via nsupdate

```python
#!/usr/bin/env python3
# sync_ipam_to_bind.py
# Dynamically update BIND DNS from NetBox using nsupdate

import subprocess
import pynetbox
import ipaddress
import dns.resolver
import dns.name

nb = pynetbox.api("http://netbox.internal", token="your-token")
DNS_SERVER = "2001:db8::53"
TSIG_KEY = "your-tsig-key"

def ipv6_to_ptr(addr: str) -> str:
    ip = ipaddress.ip_address(addr)
    full_hex = ip.exploded.replace(':', '')
    return '.'.join(reversed(full_hex)) + '.ip6.arpa.'

def get_current_aaaa(hostname: str) -> str | None:
    try:
        answers = dns.resolver.resolve(hostname, 'AAAA')
        return str(answers[0])
    except:
        return None

def update_dns_record(hostname: str, ipv6_addr: str):
    """Add or update AAAA and PTR records using nsupdate."""
    ptr_name = ipv6_to_ptr(ipv6_addr)

    nsupdate_script = f"""
server {DNS_SERVER}
zone example.com
update delete {hostname}. AAAA
update add {hostname}. 300 AAAA {ipv6_addr}
send

zone ip6.arpa
update delete {ptr_name} PTR
update add {ptr_name} 300 PTR {hostname}.
send
"""

    result = subprocess.run(
        ["nsupdate", "-k", TSIG_KEY, "-v"],
        input=nsupdate_script.encode(),
        capture_output=True
    )

    if result.returncode == 0:
        print(f"Updated DNS: {hostname} AAAA {ipv6_addr}")
    else:
        print(f"DNS update FAILED for {hostname}: {result.stderr.decode()}")

# Sync all NetBox addresses to DNS
for ip in nb.ipam.ip_addresses.filter(family=6, status="active"):
    if not ip.dns_name:
        continue

    hostname = str(ip.dns_name).rstrip('.')
    addr = str(ip.address).split('/')[0]

    # Check if DNS already matches
    current = get_current_aaaa(hostname)
    if current != addr:
        update_dns_record(hostname, addr)
    else:
        print(f"DNS already correct: {hostname} -> {addr}")
```

## PowerDNS Integration via API

```python
#!/usr/bin/env python3
# ipam_to_powerdns.py

import requests
import pynetbox
import ipaddress

nb = pynetbox.api("http://netbox.internal", token="your-token")
PDNS_URL = "http://[::1]:8081/api/v1/servers/localhost"
PDNS_KEY = "your-pdns-api-key"
HEADERS = {"X-API-Key": PDNS_KEY, "Content-Type": "application/json"}

def ipv6_to_ptr(addr: str) -> str:
    ip = ipaddress.ip_address(addr)
    full_hex = ip.exploded.replace(':', '')
    return '.'.join(reversed(full_hex)) + '.ip6.arpa.'

def upsert_aaaa_record(fqdn: str, ipv6_addr: str, zone: str):
    """Create or update an AAAA record in PowerDNS."""
    rrset = {
        "rrsets": [{
            "name": fqdn + ".",
            "type": "AAAA",
            "ttl": 300,
            "changetype": "REPLACE",
            "records": [{"content": ipv6_addr, "disabled": False}]
        }]
    }
    resp = requests.patch(
        f"{PDNS_URL}/zones/{zone}",
        json=rrset, headers=HEADERS, verify=False
    )
    if resp.status_code == 204:
        print(f"Updated AAAA: {fqdn} -> {ipv6_addr}")
    else:
        print(f"Failed: {resp.status_code} {resp.text}")

# Sync
for ip in nb.ipam.ip_addresses.filter(family=6, status="active"):
    if ip.dns_name:
        addr = str(ip.address).split('/')[0]
        fqdn = str(ip.dns_name).rstrip('.')
        zone = '.'.join(fqdn.split('.')[1:]) + '.'
        upsert_aaaa_record(fqdn, addr, zone)
```

## Validation Script

```python
#!/usr/bin/env python3
# validate_ipam_dns_sync.py
# Verify that IPAM and DNS are in sync

import pynetbox
import dns.resolver

nb = pynetbox.api("http://netbox.internal", token="your-token")

mismatches = 0
for ip in nb.ipam.ip_addresses.filter(family=6, status="active"):
    if not ip.dns_name:
        continue

    ipam_addr = str(ip.address).split('/')[0]
    hostname = str(ip.dns_name).rstrip('.')

    try:
        answers = dns.resolver.resolve(hostname, 'AAAA')
        dns_addrs = [str(a) for a in answers]
        if ipam_addr not in dns_addrs:
            print(f"MISMATCH: {hostname}")
            print(f"  IPAM: {ipam_addr}")
            print(f"  DNS:  {', '.join(dns_addrs)}")
            mismatches += 1
    except dns.resolver.NXDOMAIN:
        print(f"MISSING DNS: {hostname} (IPAM: {ipam_addr})")
        mismatches += 1
    except Exception as e:
        print(f"ERROR checking {hostname}: {e}")

print(f"\nSync status: {mismatches} mismatches found")
```

## Conclusion

IPAM-DNS integration automates the most error-prone part of IPv6 address management - keeping AAAA and PTR records synchronized with assignments. Use nsupdate or the PowerDNS API for dynamic updates triggered by IPAM changes. Run a periodic validation script to detect and alert on drift between IPAM and DNS. The most important practice is making IPAM the source of truth: address assignments happen in IPAM first, and DNS records are derived from those assignments, never the other way around.
