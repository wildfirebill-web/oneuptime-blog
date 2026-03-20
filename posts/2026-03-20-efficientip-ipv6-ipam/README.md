# How to Configure EfficientIP for IPv6 IPAM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, EfficientIP, IPAM, SOLIDserver, DDI

Description: Configure EfficientIP SOLIDserver for IPv6 IPAM including space management, subnet creation, DHCPv6 configuration, and automation via the EfficientIP REST API.

## Introduction

EfficientIP SOLIDserver is an enterprise DDI platform with comprehensive IPv6 support including smart folder-based organization, DHCPv6 server, DNS AAAA record management, and REST API automation. This guide covers IPv6 IPAM configuration using the SOLIDserver REST API.

## Step 1: Create IPv6 Space

EfficientIP organizes addresses under "Spaces" (equivalent to VRFs or address domains):

```bash
# Create IPv6 space via REST API

curl -u admin:password \
    -H "Content-Type: application/json" \
    -X POST \
    "https://efficientip.example.com/rest/ip6_block6_add" \
    -d '{
        "site_name": "Default",
        "subnet6_addr": "2001:db8::",
        "subnet6_prefix": "32",
        "subnet6_name": "Org IPv6 Allocation",
        "subnet6_class_name": "block"
    }'
```

## Step 2: Python REST API Integration

```python
#!/usr/bin/env python3
# efficientip_ipv6.py

import requests
import base64

EIP_URL = "https://efficientip.example.com/rest"
CREDS = base64.b64encode(b"admin:password").decode()
HEADERS = {
    "Authorization": f"Basic {CREDS}",
    "Content-Type": "application/json"
}

def eip_get(endpoint, params=None):
    resp = requests.get(f"{EIP_URL}/{endpoint}",
                         params=params, headers=HEADERS, verify=False)
    resp.raise_for_status()
    return resp.json()

def eip_post(endpoint, data):
    resp = requests.post(f"{EIP_URL}/{endpoint}",
                          json=data, headers=HEADERS, verify=False)
    resp.raise_for_status()
    return resp.json()

# Create /48 network
eip_post("ip6_subnet6_add", {
    "site_name": "Default",
    "subnet6_addr": "2001:db8:0001::",
    "subnet6_prefix": "48",
    "subnet6_name": "HQ Site",
    "subnet6_class_name": "network",
    "subnet6_class_parameters": "site=headquarters|environment=production"
})

# Create /64 VLAN subnet
eip_post("ip6_subnet6_add", {
    "site_name": "Default",
    "subnet6_addr": "2001:db8:0001:0001::",
    "subnet6_prefix": "64",
    "subnet6_name": "HQ Servers",
    "subnet6_class_name": "vlan",
    "subnet6_class_parameters": "vlan_id=10|gateway=2001:db8:0001:0001::1"
})
```

## Step 3: Configure DHCPv6 Server

```python
# Add DHCPv6 range to a /64 subnet
eip_post("dhcp6_range6_add", {
    "site_name": "Default",
    "dhcpscope6_name": "HQ-Servers-Scope",
    "dhcprange6_start_addr": "2001:db8:0001:0001::1000",
    "dhcprange6_end_addr": "2001:db8:0001:0001::9fff",
    "dhcprange6_class_parameters": "lease_time=3600"
})

# Configure DHCPv6 options for a scope
eip_post("dhcp6_failover6_add", {
    "dhcpserver6_name": "primary-dhcpv6",
    "failover6_type": "primary",
    "failover6_peer": "secondary-dhcpv6.internal"
})
```

## Step 4: DNS AAAA Records

```python
# Create AAAA record
eip_post("dns_rr_add", {
    "dnszone_name": "example.com",
    "rr_name": "server-01",
    "rr_type": "AAAA",
    "value1": "2001:db8:0001:0001::10",
    "rr_ttl": "300"
})

# Batch-create AAAA records from allocation list
servers = [
    ("web-01", "2001:db8:0001:0001::10"),
    ("web-02", "2001:db8:0001:0001::11"),
    ("api-01", "2001:db8:0001:0001::20"),
    ("db-01",  "2001:db8:0001:0001::30"),
]

for name, addr in servers:
    eip_post("dns_rr_add", {
        "dnszone_name": "example.com",
        "rr_name": name,
        "rr_type": "AAAA",
        "value1": addr,
        "rr_ttl": "300",
        "dns_name": f"{name}.example.com"
    })
    print(f"Created AAAA: {name}.example.com -> {addr}")
```

## Step 5: Query IPv6 Utilization

```python
# Get subnet utilization report
subnets = eip_get("ip6_subnet6_list", {
    "WHERE": "subnet6_addr LIKE '2001:db8:%'",
    "ORDERBY": "subnet6_addr"
})

print(f"{'Subnet':<35} {'Name':<20} {'Util%':>6}")
print("-" * 65)
for subnet in subnets:
    prefix = f"{subnet['subnet6_addr']}/{subnet['subnet6_prefix']}"
    name = subnet.get("subnet6_name", "")
    util = subnet.get("subnet6_utilization", "0")
    print(f"{prefix:<35} {name:<20} {util:>6}%")
```

## Step 6: IP Address Assignment

```python
# Assign next available IPv6 in a subnet
def get_next_available_ipv6(subnet_addr: str, prefix: str) -> str:
    result = eip_get("ip6_find_free_address6", {
        "site_name": "Default",
        "subnet6_addr": subnet_addr,
        "subnet6_prefix": prefix,
        "max_find": "1"
    })
    return result[0]["hostaddr"]

next_ip = get_next_available_ipv6("2001:db8:0001:0001::", "64")
eip_post("ip6_address6_add", {
    "site_name": "Default",
    "hostaddr": next_ip,
    "name": "app-server-05",
    "mac_addr": "aa:bb:cc:dd:ee:ff"
})
```

## Conclusion

EfficientIP SOLIDserver provides comprehensive IPv6 DDI management through its `ip6_subnet6_*`, `dhcp6_range6_*`, and `dns_rr_add` REST API endpoints. Its class parameter system (`subnet6_class_parameters`) enables rich metadata like VLAN ID, gateway, and environment tagging on IPv6 subnets. The `ip6_find_free_address6` endpoint enables conflict-free automated allocation. EfficientIP is particularly strong for organizations that require integrated DNS and DHCPv6 management alongside IPAM, as all three are managed through the same API and database.
