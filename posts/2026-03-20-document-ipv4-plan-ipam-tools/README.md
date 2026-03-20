# How to Document an IPv4 Address Plan with IPAM Tools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPAM, IPv4, NetBox, phpIPAM, Documentation, Network Management

Description: Learn how to document and manage your IPv4 address plan using IPAM (IP Address Management) tools like NetBox and phpIPAM, including API-based automation.

## Why Use IPAM Tools?

A spreadsheet IPv4 plan quickly becomes unmanageable at scale. IPAM tools provide:
- Visual subnet hierarchy and utilization
- Conflict detection and validation
- API-based automation
- Change history and audit trail
- Integration with DNS and DHCP

## Step 1: Install NetBox (Docker)

```bash
git clone https://github.com/netbox-community/netbox-docker.git
cd netbox-docker

# Configure

cp env/netbox.env.example env/netbox.env
# Edit env/netbox.env: set ALLOWED_HOSTS, SECRET_KEY

# Start
docker compose up -d

# Access at http://localhost:8080
# Default credentials: admin / admin
```

## Step 2: Create Prefix Hierarchy in NetBox

Via the web UI:
1. IPAM → Prefixes → Add
2. Create parent blocks first (10.0.0.0/8)
3. Then add regional blocks (10.1.0.0/16)
4. Then campus/site blocks (10.1.0.0/21)
5. Then VLAN/floor subnets (10.1.0.0/24)

## Step 3: Automate with NetBox API

```python
import requests

NETBOX_URL = "http://localhost:8080"
NETBOX_TOKEN = "your_api_token_here"

headers = {
    "Authorization": f"Token {NETBOX_TOKEN}",
    "Content-Type": "application/json",
    "Accept": "application/json",
}

def create_prefix(prefix, description, status="active", site=None):
    """Create a prefix in NetBox."""
    data = {
        "prefix": prefix,
        "description": description,
        "status": status,
    }
    if site:
        data["site"] = {"slug": site}

    response = requests.post(
        f"{NETBOX_URL}/api/ipam/prefixes/",
        json=data,
        headers=headers,
    )
    response.raise_for_status()
    return response.json()

def get_prefix_utilization(prefix_id):
    """Get utilization percentage of a prefix."""
    response = requests.get(
        f"{NETBOX_URL}/api/ipam/prefixes/{prefix_id}/",
        headers=headers,
    )
    data = response.json()
    return {
        'prefix': data['prefix'],
        'utilization': data.get('utilization', 0),
        'children': data.get('children', 0),
    }

# Create the addressing hierarchy
prefixes = [
    ("10.0.0.0/8", "Enterprise Root"),
    ("10.1.0.0/16", "Americas Region"),
    ("10.1.0.0/21", "NYC HQ Campus"),
    ("10.1.0.0/24", "NYC Floor 1 - Reception"),
    ("10.1.1.0/24", "NYC Floor 2 - Sales"),
]

for prefix, description in prefixes:
    result = create_prefix(prefix, description)
    print(f"Created: {result['prefix']} (ID: {result['id']})")
```

## Step 4: Import Existing Address Plan

```python
import requests
import csv

NETBOX_URL = "http://localhost:8080"
NETBOX_TOKEN = "your_token"

headers = {"Authorization": f"Token {NETBOX_TOKEN}", "Content-Type": "application/json"}

def import_from_csv(csv_file):
    """Import prefixes from a CSV file to NetBox."""
    # CSV format: prefix,description,site,vlan,status
    with open(csv_file) as f:
        reader = csv.DictReader(f)
        for row in reader:
            data = {
                "prefix": row['prefix'],
                "description": row['description'],
                "status": row.get('status', 'active'),
            }

            response = requests.post(
                f"{NETBOX_URL}/api/ipam/prefixes/",
                json=data,
                headers=headers,
            )

            if response.status_code == 201:
                print(f"✓ {row['prefix']}: {row['description']}")
            elif response.status_code == 400 and "already exists" in response.text:
                print(f"EXISTS: {row['prefix']}")
            else:
                print(f"✗ {row['prefix']}: {response.text}")

import_from_csv('ipv4-plan.csv')
```

## Step 5: Check for Available Subnets

```python
def get_available_subnets(parent_prefix, desired_prefix_length):
    """Find available subnets within a parent prefix."""
    parent_id_response = requests.get(
        f"{NETBOX_URL}/api/ipam/prefixes/?prefix={parent_prefix}",
        headers=headers,
    )
    prefixes = parent_id_response.json()['results']
    if not prefixes:
        print(f"Parent prefix {parent_prefix} not found")
        return

    parent_id = prefixes[0]['id']

    available_response = requests.get(
        f"{NETBOX_URL}/api/ipam/prefixes/{parent_id}/available-prefixes/",
        headers=headers,
    )

    available = available_response.json()
    print(f"Available prefixes within {parent_prefix}:")
    for avail in available[:10]:
        print(f"  {avail['prefix']} (size: {avail['family']['value']})")

get_available_subnets('10.1.0.0/16', 24)
```

## Step 6: phpIPAM as an Alternative

For simpler deployments, phpIPAM is lighter weight:

```bash
# Install phpIPAM with Docker
docker run -d \
  --name phpipam \
  -p 80:80 \
  -e MYSQL_ROOT_PASSWORD=secret \
  phpipam/phpipam-www:latest

# Access at http://localhost
# API available at http://localhost/api/
```

## Conclusion

IPAM tools like NetBox and phpIPAM bring structure to IPv4 address management. NetBox's prefix hierarchy, utilization tracking, and REST API make it ideal for large enterprises. Import your existing plan via CSV or API, create sites and VLANs to associate prefixes with network topology, and use the API for automation - creating prefixes when new VLANs are provisioned and marking IPs as used when devices are deployed. The investment in IPAM documentation pays dividends during troubleshooting and capacity planning.
