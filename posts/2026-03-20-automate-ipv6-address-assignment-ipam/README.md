# How to Automate IPv6 Address Assignment with IPAM APIs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPAM, Automation, Ansible, Terraform, REST API

Description: Automate IPv6 address assignment using IPAM REST APIs with Ansible, Terraform, and Python scripts to provision addresses for new infrastructure without manual IPAM updates.

## Introduction

Manual IPAM updates are the main source of IPv6 address conflicts and stale records. Automating address assignment through IPAM APIs ensures every address is allocated through the IPAM system, preventing duplicate assignments and keeping documentation current. This guide covers automation via Python, Ansible, and Terraform.

## Python: Generic IPAM Allocation Function

```python
#!/usr/bin/env python3
# ipam_allocate.py

import pynetbox
import ipaddress
from typing import Optional

nb = pynetbox.api("http://netbox.internal", token="your-token")

def allocate_ipv6_address(
    prefix: str,
    hostname: str,
    description: str = "",
    dns_name: str = "",
    role: Optional[str] = None
) -> str:
    """
    Allocate the next available IPv6 /128 from a prefix.
    Creates the address record in NetBox and returns the address string.
    """
    # Find the prefix object
    prefix_obj = nb.ipam.prefixes.get(prefix=prefix)
    if not prefix_obj:
        raise ValueError(f"Prefix {prefix} not found in NetBox")

    # Request next available IP from the prefix
    available = nb.ipam.prefixes.available_ips.create(
        prefix_obj.id,
        {
            "description": description or f"Auto-allocated for {hostname}",
            "dns_name": dns_name or f"{hostname}.example.com",
            "status": "active",
            "tags": [{"slug": "auto-allocated"}]
        }
    )

    allocated_addr = available["address"]
    print(f"Allocated: {allocated_addr} for {hostname}")
    return allocated_addr

# Usage

ip = allocate_ipv6_address(
    prefix="2001:db8:0001:0001::/64",
    hostname="app-server-10",
    description="Application server",
    dns_name="app-server-10.internal.example.com"
)
print(f"New server will use: {ip}")
```

## Ansible Module Usage

```yaml
# playbooks/provision_server.yml

- name: Provision new server with IPv6 address
  hosts: localhost
  vars:
    netbox_url: "http://netbox.internal"
    netbox_token: "{{ lookup('env', 'NETBOX_TOKEN') }}"
    server_name: "app-server-10"
    target_prefix: "2001:db8:0001:0001::/64"

  tasks:
    - name: Get next available IPv6 in prefix
      netbox.netbox.netbox_available_ip:
        netbox_url: "{{ netbox_url }}"
        netbox_token: "{{ netbox_token }}"
        parent: "{{ target_prefix }}"
        data:
          description: "Auto-allocated for {{ server_name }}"
          dns_name: "{{ server_name }}.example.com"
          status: active
      register: ipv6_allocation

    - name: Display allocated address
      debug:
        msg: "Allocated IPv6: {{ ipv6_allocation.ip_address.address }}"

    - name: Store IPv6 in inventory
      add_host:
        name: "{{ server_name }}"
        ansible_host: "{{ ipv6_allocation.ip_address.address | ipaddr('address') }}"
```

## Terraform Integration

```hcl
# main.tf - Terraform with NetBox provider

terraform {
  required_providers {
    netbox = {
      source  = "e-breuninger/netbox"
      version = "~> 3.0"
    }
  }
}

provider "netbox" {
  server_url = "http://netbox.internal"
  api_token  = var.netbox_token
}

# Allocate next available IPv6 from a prefix
resource "netbox_available_ip_address" "server_ipv6" {
  prefix_id   = data.netbox_prefix.servers.id
  dns_name    = "app-server-10.example.com"
  description = "Terraform-provisioned server"
  status      = "active"

  lifecycle {
    # Prevent accidental address change
    ignore_changes = [prefix_id]
  }
}

data "netbox_prefix" "servers" {
  prefix = "2001:db8:0001:0001::/64"
}

output "server_ipv6" {
  value = netbox_available_ip_address.server_ipv6.ip_address
}
```

## CI/CD Integration

```bash
#!/bin/bash
# allocate_and_deploy.sh
# Allocate IPv6 address during CI/CD pipeline and deploy

SERVER_NAME="${1:-new-server}"
PREFIX="2001:db8:0001:0001::/64"

# Allocate from NetBox via API
ALLOC=$(curl -s \
    -H "Authorization: Token $NETBOX_TOKEN" \
    -H "Content-Type: application/json" \
    -X POST \
    "http://netbox.internal/api/ipam/prefixes/$(get_prefix_id "$PREFIX")/available-ips/" \
    -d "{\"description\": \"CI/CD allocation for $SERVER_NAME\", \"status\": \"active\"}")

IPv6_ADDR=$(echo "$ALLOC" | python3 -c "import json,sys; print(json.load(sys.stdin)['address'].split('/')[0])")
echo "Allocated IPv6: $IPv6_ADDR"

# Use the address in deployment
ansible-playbook playbooks/deploy_server.yml \
    -e "server_name=$SERVER_NAME" \
    -e "server_ipv6=$IPv6_ADDR"
```

## Address Lifecycle Management

```python
def deallocate_ipv6_address(ipv6_addr: str, reason: str = ""):
    """Mark an IPv6 address as available (decommission workflow)."""
    # Find the IP address object
    ip_obj = nb.ipam.ip_addresses.get(address=ipv6_addr)
    if not ip_obj:
        print(f"Address {ipv6_addr} not found in NetBox")
        return

    # Update status to deprecated first (soft delete)
    nb.ipam.ip_addresses.update([{
        "id": ip_obj.id,
        "status": "deprecated",
        "description": f"DECOMMISSIONED: {reason}"
    }])

    print(f"Marked {ipv6_addr} as deprecated: {reason}")

# Usage in decommission workflow
deallocate_ipv6_address("2001:db8:0001:0001::10", "Server retired 2026-03-20")
```

## Conclusion

Automating IPv6 address assignment through IPAM APIs (NetBox, Infoblox, etc.) eliminates manual allocation errors and ensures every address is tracked from the moment it is assigned. Use the `available-ips` endpoint (NetBox) or equivalent to atomically allocate the next available address without conflict checking in your automation code - the IPAM system handles concurrency. Integrate allocation into CI/CD pipelines so infrastructure provisioned through automation always updates IPAM, and build decommission workflows that mark addresses as deprecated rather than deleting them to preserve audit history.
