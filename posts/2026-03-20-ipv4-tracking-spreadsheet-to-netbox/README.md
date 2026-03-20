# How to Migrate from Spreadsheet-Based IPv4 Tracking to NetBox or phpIPAM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPAM, NetBox, phpIPAM, IPv4, Network Management, Migration

Description: Migrate your IPv4 address inventory from spreadsheets to a proper IPAM tool like NetBox or phpIPAM using bulk import via CSV and API automation.

## Introduction

Many organizations track IP addresses in Excel or Google Sheets. While this works for small networks, it breaks down at scale - no conflict detection, no audit trail, no API integration. Migrating to NetBox (open-source, API-first) or phpIPAM gives you a proper IPAM system with automation capabilities.

## Why Migrate to a Proper IPAM Tool

| Feature | Spreadsheet | NetBox/phpIPAM |
|---------|-------------|----------------|
| Duplicate detection | Manual | Automatic |
| Subnet visualization | No | Yes |
| API for automation | No | Full REST API |
| Change audit trail | No | Built-in |
| VLAN management | Limited | Full |
| DNS/DHCP integration | No | Yes |

## Step 1: Export Your Spreadsheet

Structure your existing data into a CSV with standard columns:

```csv
# ip-inventory.csv

ip_address,subnet,hostname,description,device,location,vlan,status
10.1.1.10,10.1.1.0/24,web-01.example.com,Web server,Dell PowerEdge,NYC-DC,10,active
10.1.1.11,10.1.1.0/24,web-02.example.com,Web server,Dell PowerEdge,NYC-DC,10,active
10.1.2.50,10.1.2.0/24,workstation-50,John Smith workstation,HP EliteBook,NYC-Office,20,active
```

## Step 2A: Import into NetBox via CSV

NetBox supports native CSV import for most object types.

```bash
# NetBox API: Create prefixes (subnets) first
curl -s -X POST "http://netbox.example.com/api/ipam/prefixes/" \
  -H "Authorization: Token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    "prefix": "10.1.1.0/24",
    "description": "NYC DC Server VLAN",
    "vlan": {"id": 10},
    "site": {"name": "New York DC"},
    "status": "active"
  }'

# Create IP address records via API (script to loop over CSV)
python3 << 'PYEOF'
import csv, requests

NETBOX_URL = "http://netbox.example.com/api"
TOKEN = "your-api-token"
HEADERS = {"Authorization": f"Token {TOKEN}", "Content-Type": "application/json"}

with open("ip-inventory.csv") as f:
    for row in csv.DictReader(f):
        data = {
            "address": f"{row['ip_address']}/24",
            "dns_name": row["hostname"],
            "description": row["description"],
            "status": row["status"]
        }
        r = requests.post(f"{NETBOX_URL}/ipam/ip-addresses/", json=data, headers=HEADERS)
        if r.status_code == 201:
            print(f"Created: {row['ip_address']}")
        else:
            print(f"Error {r.status_code}: {row['ip_address']} - {r.text}")
PYEOF
```

## Step 2B: Import into phpIPAM via CSV

phpIPAM supports CSV import from the UI:

1. Navigate to **IP Addresses > Import addresses**
2. Upload your CSV file
3. Map CSV columns to phpIPAM fields
4. Review conflicts and import

Or use the API:

```bash
#!/bin/bash
# Bulk import via phpIPAM API

TOKEN=$(curl -s -X POST "http://phpipam.example.com/api/myapp/user/" \
  -u "admin:password" | jq -r '.data.token')

while IFS=, read -r ip subnet hostname description rest; do
    [[ "$ip" == "ip_address" ]] && continue   # Skip header
    curl -s -X POST "http://phpipam.example.com/api/myapp/addresses/" \
      -H "Content-Type: application/json" \
      -H "token: ${TOKEN}" \
      -d "{\"subnetId\": \"5\", \"ip\": \"${ip}\", \"hostname\": \"${hostname}\", \"description\": \"${description}\"}"
done < ip-inventory.csv
```

## Step 3: Validate the Migration

After import, validate the data:

```bash
# NetBox: Check for IP addresses without a prefix
curl -s "http://netbox.example.com/api/ipam/ip-addresses/?limit=1000" \
  -H "Authorization: Token your-api-token" | \
  jq '.results[] | select(.prefix == null) | .address'

# Check for duplicates
curl -s "http://netbox.example.com/api/ipam/ip-addresses/?limit=1000" \
  -H "Authorization: Token your-api-token" | \
  jq '[.results[].address] | group_by(.) | map(select(length > 1)) | .[]'
```

## Step 4: Set Up Ongoing Automation

```bash
# Configure DHCP server integration (ISC DHCP/Kea) to update IPAM on lease events
# Configure your provisioning tool (Terraform/Ansible) to allocate IPs from IPAM
# Set up periodic scanning to detect unregistered IPs
```

## Conclusion

Migrating to NetBox or phpIPAM replaces fragile spreadsheets with a robust, API-driven IPAM platform. The bulk import process is straightforward - export to CSV, transform to the tool's format, and validate after import. Once live, integrate with your provisioning workflows to keep the IPAM database accurate automatically.
