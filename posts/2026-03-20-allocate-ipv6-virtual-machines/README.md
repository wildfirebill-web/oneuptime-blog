# How to Allocate IPv6 Addresses to Virtual Machines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPAM, Virtual Machines, Address Allocation, DHCPv6, SLAAC

Description: Implement IPv6 address allocation strategies for virtual machines using SLAAC, DHCPv6 reservations, and IPAM integration to ensure unique, trackable IPv6 addresses for each VM.

## Introduction

Allocating IPv6 addresses to virtual machines requires choosing between SLAAC (automatic from MAC), DHCPv6 (assigned by server), or static assignment. Each approach has trade-offs for address predictability, IPAM tracking, and operational complexity. This guide covers practical allocation strategies for VM environments, including IPAM integration.

## Address Allocation Strategies

```
Strategy 1: SLAAC (Stateless Address Autoconfiguration)
├─ VM generates address from MAC using EUI-64
├─ Predictable if MAC is known
├─ No DHCPv6 server needed
└─ Hard to track in IPAM without MAC→IP correlation

Strategy 2: DHCPv6 Stateful
├─ DHCPv6 server assigns address from pool
├─ Can reserve specific address per VM (by DUID)
├─ IPAM records DHCPv6 leases
└─ Requires DHCPv6 server infrastructure

Strategy 3: Static (IPAM-assigned)
├─ IPAM allocates next available /128 from prefix
├─ Passed to VM via cloud-init, Terraform, or Ansible
├─ Most traceable — IPAM owns the record
└─ Requires automation for scalability
```

## SLAAC: Predict IPv6 from MAC Address

```python
#!/usr/bin/env python3
# slaac_from_mac.py

import ipaddress

def eui64_from_mac(mac: str) -> str:
    """Generate EUI-64 identifier from MAC address."""
    mac_bytes = [int(b, 16) for b in mac.split(":")]
    # Insert ff:fe in the middle
    eui64 = mac_bytes[:3] + [0xff, 0xfe] + mac_bytes[3:]
    # Flip the U/L bit (7th bit of first byte)
    eui64[0] ^= 0x02
    return ":".join(f"{b:02x}" for b in eui64)

def slaac_address(prefix: str, mac: str) -> str:
    """Compute SLAAC IPv6 address for a prefix and MAC."""
    net = ipaddress.ip_network(prefix)
    eui64 = eui64_from_mac(mac)
    # Combine /64 prefix with EUI-64 interface ID
    prefix_int = int(net.network_address)
    iid_int = int(ipaddress.ip_address(f"::{''.join(eui64.split(':'))}"))
    addr = ipaddress.ip_address(prefix_int | iid_int)
    return str(addr)

# Example: VM with MAC 52:54:00:ab:cd:01 on prefix 2001:db8:vms::/64
mac = "52:54:00:ab:cd:01"
prefix = "2001:db8:vms::/64"
ipv6 = slaac_address(prefix, mac)
print(f"SLAAC address: {ipv6}")
# Output: 2001:db8:vms::5054:ff:feab:cd01
```

## DHCPv6 Reservations by DUID

```bash
# ISC DHCP / Kea: reserve IPv6 by DUID (linked to VM's identity)

# Kea DHCPv6 reservation by DUID
# /etc/kea/kea-dhcp6.conf
cat >> /etc/kea/kea-dhcp6.conf << 'EOF'
{
  "Dhcp6": {
    "reservations": [
      {
        "hw-address": "52:54:00:ab:cd:01",
        "ip-addresses": ["2001:db8:vms::10"],
        "hostname": "myvm1"
      },
      {
        "hw-address": "52:54:00:ab:cd:02",
        "ip-addresses": ["2001:db8:vms::11"],
        "hostname": "myvm2"
      }
    ]
  }
}
EOF

systemctl restart kea-dhcp6-server
```

## NetBox IPAM Integration for VM IPv6

```python
#!/usr/bin/env python3
# vm_ipv6_allocator.py

import pynetbox
import ipaddress

nb = pynetbox.api("https://netbox.example.com", token="your-token")

def allocate_ipv6_for_vm(vm_name: str, prefix_id: int, dns_name: str = "") -> str:
    """Allocate next available IPv6 from a NetBox prefix for a VM."""

    # Get next available IP from the prefix
    prefix = nb.ipam.prefixes.get(prefix_id)
    available = prefix.available_ips.create({
        "status": "active",
        "dns_name": dns_name,
        "description": f"Allocated to VM: {vm_name}",
    })

    ipv6_address = available["address"]

    # Create or update the VM record with the allocated IP
    vms = nb.virtualization.virtual_machines.filter(name=vm_name)
    if vms:
        vm = vms[0]
        # Add the IP address to the VM's primary_ip6
        ip_obj = nb.ipam.ip_addresses.get(address=ipv6_address)
        if ip_obj:
            nb.virtualization.virtual_machines.update([{
                "id": vm.id,
                "primary_ip6": ip_obj.id,
            }])

    print(f"Allocated {ipv6_address} to VM {vm_name}")
    return ipv6_address

# Example usage
ipv6 = allocate_ipv6_for_vm(
    vm_name="webserver-01",
    prefix_id=42,  # NetBox prefix ID for 2001:db8:vms::/64
    dns_name="webserver-01.example.com",
)
```

## Terraform: Allocate IPv6 from NetBox

```hcl
# main.tf

terraform {
  required_providers {
    netbox = {
      source  = "e-breuninger/netbox"
      version = "~> 3.0"
    }
  }
}

provider "netbox" {
  server_url = "https://netbox.example.com"
  api_token  = var.netbox_token
}

# Allocate IPv6 for each VM
resource "netbox_available_ip_address" "vm_ipv6" {
  count     = var.vm_count
  prefix_id = var.ipv6_prefix_id    # NetBox prefix for VM allocation
  status    = "active"
  dns_name  = "vm-${count.index}.example.com"
  description = "VM ${count.index} IPv6 address"
}

output "vm_ipv6_addresses" {
  value = netbox_available_ip_address.vm_ipv6[*].ip_address
}

# Use the allocated IPs in VM definitions
resource "libvirt_domain" "vm" {
  count = var.vm_count
  name  = "vm-${count.index}"
  memory = 2048

  network_interface {
    network_name = "default"
  }

  # Configure IPv6 via cloud-init using allocated address
  cloudinit = libvirt_cloudinit_disk.vm_init[count.index].id
}

resource "libvirt_cloudinit_disk" "vm_init" {
  count = var.vm_count
  name  = "vm-${count.index}-init.iso"
  user_data = templatefile("cloud-init.yaml.tpl", {
    ipv6_address = netbox_available_ip_address.vm_ipv6[count.index].ip_address
    ipv6_gateway = var.ipv6_gateway
  })
}
```

## Cloud-init: Apply Allocated IPv6 to VM

```yaml
# cloud-init.yaml.tpl (Terraform template)
#cloud-config
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      addresses:
        - "${ipv6_address}"
      gateway6: "${ipv6_gateway}"
      nameservers:
        addresses:
          - "2001:db8::53"
```

## Bulk Allocation Script

```python
#!/usr/bin/env python3
# bulk_allocate_ipv6.py

import pynetbox
import csv

nb = pynetbox.api("https://netbox.example.com", token="your-token")

def bulk_allocate_from_csv(csv_file: str, prefix_id: int) -> list:
    """Allocate IPv6 addresses for VMs listed in CSV."""
    results = []
    prefix = nb.ipam.prefixes.get(prefix_id)

    with open(csv_file) as f:
        reader = csv.DictReader(f)
        for row in reader:
            vm_name = row["vm_name"]
            allocated = prefix.available_ips.create({
                "status": "active",
                "dns_name": f"{vm_name}.example.com",
                "description": f"VM: {vm_name}",
            })
            results.append({
                "vm_name": vm_name,
                "ipv6": allocated["address"],
            })
            print(f"  {vm_name}: {allocated['address']}")

    return results

# Usage: bulk_allocate_from_csv("vms.csv", prefix_id=42)
```

## Conclusion

IPv6 address allocation for VMs involves three approaches: SLAAC (predictable from MAC, no server needed), DHCPv6 reservations (by DUID/MAC in Kea or ISC DHCP), and IPAM-driven static assignment (most traceable). The SLAAC EUI-64 algorithm is deterministic — knowing a VM's MAC lets you predict its SLAAC address for a given /64 prefix. For production environments, IPAM integration via NetBox or similar tools provides centralized tracking of which VM holds which IPv6 address. Terraform and cloud-init work together for automated allocation: Terraform queries the IPAM API for the next available IPv6, then passes it to cloud-init for configuration during VM boot.
