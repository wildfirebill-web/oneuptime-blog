# How to Set Up IPv4 Address Scanning and Discovery with IPAM Tools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPAM, IPv4, Network Discovery, Scanning, nmap, NetBox

Description: Configure automated IPv4 address scanning with nmap and integrate discovered hosts into IPAM tools for accurate inventory management.

Network discovery scanning finds IPv4 addresses that are in use, identifies undocumented hosts, and reconciles discovered devices with your IPAM inventory.

## Basic Network Discovery with nmap

```bash
# Install nmap

sudo apt install nmap -y

# Discover live hosts in a subnet (ping scan)
sudo nmap -sn 10.100.1.0/24

# Output:
# Nmap scan report for 10.100.1.1
# Host is up (0.0012s latency).
# ...
# Nmap done: 256 IP addresses (12 hosts up) scanned in 2.30 seconds

# Get live hosts only (no hostnames)
sudo nmap -sn 10.100.1.0/24 | grep "report for" | awk '{print $5}'
```

## Comprehensive Host Discovery

```bash
# More thorough discovery: ARP scan (same subnet), ICMP, TCP port 80/443
sudo nmap -sn -PE -PP -PS80,443 -PA3389 10.100.1.0/24

# With OS detection and version scanning (requires root)
sudo nmap -sV -O --open 10.100.1.0/24 -oX scan_results.xml

# Output as greppable format
sudo nmap -sn 10.100.0.0/22 -oG - | grep "Up" | awk '{print $2}'
```

## Python Script: Scan and Compare with NetBox

```python
#!/usr/bin/env python3
# scan_and_reconcile.py
# Scan a subnet and compare discovered IPs with NetBox records

import subprocess
import requests
import json

NETBOX_URL = "http://netbox.example.com"
TOKEN = "your-api-token"
SUBNET = "10.100.1.0/24"

def get_live_ips(subnet):
    """Run nmap and return list of live IPv4 addresses."""
    result = subprocess.run(
        ["nmap", "-sn", subnet, "-oG", "-"],
        capture_output=True, text=True
    )
    live_ips = []
    for line in result.stdout.splitlines():
        if "Up" in line and "Host:" in line:
            ip = line.split()[1]
            live_ips.append(ip)
    return live_ips

def get_netbox_ips(prefix):
    """Get all documented IPs from NetBox for a prefix."""
    resp = requests.get(
        f"{NETBOX_URL}/api/ipam/ip-addresses/",
        headers={"Authorization": f"Token {TOKEN}"},
        params={"parent": prefix, "limit": 1000}
    )
    return {ip["address"].split("/")[0]: ip for ip in resp.json()["results"]}

def scan_and_reconcile(subnet):
    print(f"Scanning {subnet}...")
    live_ips = get_live_ips(subnet)
    documented = get_netbox_ips(subnet)

    undocumented = [ip for ip in live_ips if ip not in documented]
    documented_but_offline = [ip for ip, info in documented.items()
                              if ip not in live_ips and info["status"]["value"] == "active"]

    print(f"\nLive hosts: {len(live_ips)}")
    print(f"Documented in NetBox: {len(documented)}")

    if undocumented:
        print(f"\nUNDOCUMENTED live hosts (not in NetBox):")
        for ip in undocumented:
            print(f"  {ip}")

    if documented_but_offline:
        print(f"\nDocumented as active but not responding:")
        for ip in documented_but_offline:
            print(f"  {ip} ({documented[ip].get('dns_name', 'no DNS name')})")

scan_and_reconcile(SUBNET)
```

## Scheduled Scanning with Cron

```bash
# /etc/cron.d/ip-scan
# Run discovery every night at 2 AM
0 2 * * * root python3 /usr/local/bin/scan_and_reconcile.py >> /var/log/ip-scan.log 2>&1
```

## Using arp-scan for Local Network Discovery

```bash
# arp-scan is faster and more reliable on local subnets
sudo apt install arp-scan -y

# Scan local subnet
sudo arp-scan --localnet

# Scan a specific subnet via a specific interface
sudo arp-scan -I eth0 10.100.1.0/24

# Output includes IP, MAC, and vendor
# 10.100.1.10  00:11:22:33:44:55  Dell Inc.
```

## phpIPAM Built-in Scanning

phpIPAM has built-in subnet scanning that can be configured to run automatically:

```bash
# Enable scanning in phpIPAM CLI
php /var/www/html/phpipam/functions/scripts/scanSubnets.php
```

In the web UI: **Administration → Scan Agents → Configure** to set scan intervals per subnet.

Regular scanning ensures your IPAM stays accurate and reveals undocumented devices or stale entries that should be cleaned up.
