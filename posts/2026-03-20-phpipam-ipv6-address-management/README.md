# How to Configure phpIPAM for IPv6 Address Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, phpIPAM, IPAM, Network Management, Open Source

Description: Configure phpIPAM to manage IPv6 subnets, assign addresses, configure DHCPv6 discovery, and use the phpIPAM API for IPv6 automation.

## Introduction

phpIPAM is a PHP-based open source IPAM solution with built-in IPv6 support including subnet scanning, DHCPv6 integration, and REST API. This guide covers adding IPv6 sections, creating subnets, enabling ping discovery for IPv6, and using the API.

## Step 1: Enable IPv6 in phpIPAM

```bash
# Install phpIPAM with Docker
docker run -d --name phpipam-web \
    -p 80:80 \
    -e IPAM_DATABASE_HOST=phpipam-db \
    -e IPAM_DATABASE_USER=phpipam \
    -e IPAM_DATABASE_PASS=phpipam \
    -e IPAM_DATABASE_NAME=phpipam \
    phpipam/phpipam-www:latest

# After setup, enable IPv6 in Administration > phpIPAM settings:
# - Enable IPv6 support: Yes
# - Enable IPv6 CIDR display: Yes
```

## Step 2: Create IPv6 Sections and Subnets

In phpIPAM UI:
1. **Sections**: Create a section called "IPv6" under Administration > Sections
2. **Subnets**: Navigate to the IPv6 section and add subnets

```php
# phpIPAM stores IPv6 subnets using the standard subnet format
# Subnet: 2001:db8:0001::/48
# Description: HQ Site
# VLAN: associate with existing VLAN
```

## Step 3: phpIPAM REST API for IPv6

```python
#!/usr/bin/env python3
# phpipam_ipv6.py

import requests
import json

PHPIPAM_URL = "http://phpipam.internal"
APP_ID = "myapp"
USERNAME = "admin"
PASSWORD = "password"

# Authenticate
auth_resp = requests.post(
    f"{PHPIPAM_URL}/api/{APP_ID}/user/",
    auth=(USERNAME, PASSWORD)
)
TOKEN = auth_resp.json()["data"]["token"]
HEADERS = {"phpipam-token": TOKEN, "Content-Type": "application/json"}

# Create an IPv6 subnet
subnet_data = {
    "subnet": "2001:db8:0001::",
    "mask": "48",
    "description": "HQ Site IPv6 Prefix",
    "sectionId": "2",  # IPv6 section ID
    "isFolder": "0",
    "pingSubnet": "1"   # Enable ICMP discovery
}

resp = requests.post(
    f"{PHPIPAM_URL}/api/{APP_ID}/subnets/",
    headers=HEADERS,
    json=subnet_data
)
print(f"Created subnet: {resp.json()}")

# Add a /64 under the /48
vlan_subnet = {
    "subnet": "2001:db8:0001:0001::",
    "mask": "64",
    "description": "HQ Servers",
    "masterSubnetId": resp.json()["id"],
    "sectionId": "2"
}
resp2 = requests.post(
    f"{PHPIPAM_URL}/api/{APP_ID}/subnets/",
    headers=HEADERS,
    json=vlan_subnet
)
print(f"Created /64: {resp2.json()}")

# Get all IPv6 addresses in a subnet
subnet_id = resp2.json()["id"]
addresses = requests.get(
    f"{PHPIPAM_URL}/api/{APP_ID}/subnets/{subnet_id}/addresses/",
    headers=HEADERS
).json()
print(f"Addresses in subnet: {addresses}")
```

## Step 4: IPv6 Subnet Scanning (Ping Discovery)

phpIPAM can scan IPv6 subnets by pinging addresses to mark them as used. This works for manually assigned addresses or DHCPv6 leases but cannot discover SLAAC addresses that block ping.

```bash
# Configure cron job for IPv6 subnet scan
# In /etc/cron.d/phpipam:
*/15 * * * * www-data php /var/www/html/phpipam/functions/scripts/pingCheck.php >/dev/null 2>&1
*/5  * * * * www-data php /var/www/html/phpipam/functions/scripts/discoveryCheck.php >/dev/null 2>&1
```

## Step 5: DHCPv6 Integration

phpIPAM can import ISC DHCPv6 or Kea DHCPv6 lease files:

```bash
# Import ISC DHCPv6 leases
# In phpIPAM: Administration > DHCP > Import DHCP leases
# Import from: /var/lib/dhcpd/dhcpd6.leases

# Or configure DHCP agent (phpipam-agent) for live sync
# /etc/phpipam/config-dhcp.conf:
[dhcp]
server = dhcpv6-server.internal
key = your-dhcp-key
type = kea
```

## Step 6: Tag IPv6 Addresses by Type

phpIPAM supports custom tags for marking address types:

```python
# Tag an address as SLAAC-derived
tag_data = {
    "ip": "2001:db8:0001:0001::1234:5678:9abc",
    "subnetId": subnet_id,
    "description": "Desktop-workstation-01 SLAAC",
    "tag": "2",          # Tag ID for "Used"
    "owner": "john.doe"
}

resp = requests.post(
    f"{PHPIPAM_URL}/api/{APP_ID}/addresses/",
    headers=HEADERS,
    json=tag_data
)
```

## Conclusion

phpIPAM provides solid IPv6 IPAM for small to medium organizations with its subnet hierarchy (sections → subnets → addresses), ICMP-based discovery for IPv6, DHCPv6 lease import, and REST API. The UI is straightforward for network administrators without programming experience. For automation-heavy environments or large-scale deployments, consider NetBox's more powerful API and data model. The key phpIPAM IPv6 limitation is that SLAAC address discovery requires addresses to respond to ping, which is often blocked in security-conscious environments.
