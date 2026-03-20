# How to Configure BlueCat for IPv6 IPAM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, BlueCat, IPAM, DDI, Enterprise Networking

Description: Configure BlueCat Address Manager for IPv6 address management including block and network creation, DHCPv6 ranges, and automation via the BlueCat REST API v2.

## Introduction

BlueCat Address Manager (BAM) is an enterprise DDI platform with strong IPv6 support. Its hierarchical object model maps well to IPv6 address hierarchies: IP Blocks contain IP Networks which contain address objects. This guide covers IPv6 configuration via the BlueCat REST API v2.

## Step 1: Create IPv6 IP Block

In BlueCat, IPv6 addresses are organized under IP Blocks (aggregates) → IP Networks (subnets) → IP Addresses:

```python
#!/usr/bin/env python3
# bluecat_ipv6.py

import requests

BAM_URL = "https://bluecat.example.com/api/v2"
AUTH = ("admin", "password")

# Login and get session token

login_resp = requests.get(
    f"{BAM_URL}/login",
    auth=AUTH, verify=False
)
TOKEN = login_resp.headers.get("BAMAuthToken")
HEADERS = {
    "Authorization": f"BAMAuthToken: {TOKEN}",
    "Content-Type": "application/json"
}

def bam_post(path, data):
    resp = requests.post(f"{BAM_URL}/{path}", json=data,
                          headers=HEADERS, verify=False)
    resp.raise_for_status()
    return resp.json()

def bam_get(path, params=None):
    resp = requests.get(f"{BAM_URL}/{path}", params=params,
                         headers=HEADERS, verify=False)
    resp.raise_for_status()
    return resp.json()

# Get the root configuration (container for all objects)
configs = bam_get("configurations")
config_id = configs[0]["id"]

# Create IPv6 Block
block = bam_post("blocks", {
    "name": "Org IPv6 Allocation",
    "address": "2001:db8::",
    "cidr": 32,
    "type": "IPv6Block",
    "configuration": {"id": config_id}
})
block_id = block["id"]
print(f"Created block: {block['id']}")
```

## Step 2: Create IPv6 Networks

```python
# Create /48 site prefix under the /32 block
network = bam_post("networks", {
    "name": "HQ Site",
    "address": "2001:db8:0001::",
    "cidr": 48,
    "type": "IPv6Network",
    "block": {"id": block_id},
    "properties": "radvdEnabled=true|dhcpv6Enabled=true"
})
net_id = network["id"]

# Create /64 VLAN subnet
vlan_net = bam_post("networks", {
    "name": "HQ Servers",
    "address": "2001:db8:0001:0001::",
    "cidr": 64,
    "type": "IPv6Network",
    "block": {"id": block_id}
})
```

## Step 3: Configure DHCPv6 Range

```python
# Create DHCPv6 range in the /64
dhcp_range = bam_post("ranges", {
    "fromAddress": "2001:db8:0001:0001::1000",
    "toAddress": "2001:db8:0001:0001::9fff",
    "network": {"id": vlan_net["id"]},
    "type": "DHCP6Range",
    "properties": "offset=0"
})
```

## Step 4: Assign IPv6 Addresses

```python
# Assign a specific IPv6 address
ip_addr = bam_post("addresses", {
    "address": "2001:db8:0001:0001::10",
    "type": "IPv6Address",
    "network": {"id": vlan_net["id"]},
    "name": "web-server-01",
    "properties": "state=STATIC|macAddress=00:11:22:33:44:55"
})

# Get next available IPv6 address in a network
next_ip = bam_get(f"networks/{vlan_net['id']}/nextAvailableAddress",
                   {"count": 1})
print(f"Next available: {next_ip[0]['address']}")
```

## Step 5: Create AAAA Records

```python
# Create AAAA record via BlueCat BAM
# First get or create the DNS zone
zone = bam_get("zones", {"name": "example.com", "type": "Zone"})
zone_id = zone[0]["id"]

# Create AAAA record
aaaa = bam_post("resourceRecords", {
    "type": "AaaaRecord",
    "name": "web-server-01",
    "address": "2001:db8:0001:0001::10",
    "zone": {"id": zone_id},
    "ttl": 300
})

# Create PTR record
ptr = bam_post("resourceRecords", {
    "type": "PtrRecord",
    "name": "0.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.0.0.1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa",
    "linkedRecord": {"id": aaaa["id"]},
    "zone": {"id": bam_get("zones", {"name": "ip6.arpa"})[0]["id"]}
})
```

## Step 6: Bulk Import IPv6 Networks

```python
# Import IPv6 networks from a CSV
import csv

with open("ipv6_networks.csv") as f:
    reader = csv.DictReader(f)
    for row in reader:
        # columns: address, cidr, name, site
        bam_post("networks", {
            "address": row["address"],
            "cidr": int(row["cidr"]),
            "name": row["name"],
            "type": "IPv6Network",
            "block": {"id": block_id}
        })
        print(f"Imported: {row['address']}/{row['cidr']} - {row['name']}")
```

## Conclusion

BlueCat Address Manager provides enterprise IPv6 IPAM through its hierarchical object model (Block → Network → Address) and REST API v2. The `nextAvailableAddress` endpoint enables automated allocation, and the linked record system keeps AAAA and PTR records synchronized. BlueCat's workflow and approval system makes it suitable for organizations that need change control around IPv6 address allocations. Use the `properties` field when creating networks to enable DHCPv6 and router advertisements directly in the IPAM record.
