# How to Configure DHCP Server Integration with IPAM Tools for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, IPAM, NetBox, phpIPAM, IPv4, Automation, Network Management

Description: Integrate ISC DHCP or Kea DHCP server with NetBox or phpIPAM to automatically track IPv4 lease allocations in your IPAM database.

## Introduction

DHCP servers allocate IP addresses dynamically, but those assignments are often invisible to IPAM tools unless integrated. By connecting your DHCP server to your IPAM, every lease becomes a tracked record — giving you real-time visibility into which devices hold which IPs.

## Approach 1: DHCP Hooks to Call phpIPAM API

ISC DHCP supports `on commit`, `on release`, and `on expiry` hooks that run scripts when lease events occur.

### ISC DHCP Configuration

```
# /etc/dhcp/dhcpd.conf — add at the end

# Hook script called when a lease is committed
on commit {
    set clientIP = binary-to-ascii(10, 8, ".", leased-address);
    set clientMAC = binary-to-ascii(16, 8, ":", substring(hardware, 1, 6));
    set clientHostname = pick-first-value(host-decl-name, option fqdn.hostname,
                          option dhcp-client-identifier, "unknown");
    execute("/etc/dhcp/scripts/update-ipam.sh", "add",
            clientIP, clientMAC, clientHostname);
}

on release {
    set clientIP = binary-to-ascii(10, 8, ".", leased-address);
    execute("/etc/dhcp/scripts/update-ipam.sh", "delete", clientIP, "", "");
}

on expiry {
    set clientIP = binary-to-ascii(10, 8, ".", leased-address);
    execute("/etc/dhcp/scripts/update-ipam.sh", "delete", clientIP, "", "");
}
```

### IPAM Update Script

```bash
#!/bin/bash
# /etc/dhcp/scripts/update-ipam.sh
# Called by ISC DHCP on lease events

ACTION=$1
IP=$2
MAC=$3
HOSTNAME=$4

PHPIPAM_URL="http://phpipam.example.com/api/myapp"
TOKEN=$(curl -s -X POST "${PHPIPAM_URL}/user/" -u "dhcp-api:dhcp-password" | jq -r '.data.token')

case "${ACTION}" in
  add)
    # Search for existing record
    RECORD_ID=$(curl -s "${PHPIPAM_URL}/addresses/search/${IP}/" \
      -H "token: ${TOKEN}" | jq -r '.data[0].id // empty')

    if [[ -n "${RECORD_ID}" ]]; then
      # Update existing record
      curl -s -X PATCH "${PHPIPAM_URL}/addresses/${RECORD_ID}/" \
        -H "Content-Type: application/json" \
        -H "token: ${TOKEN}" \
        -d "{\"hostname\": \"${HOSTNAME}\", \"mac\": \"${MAC}\", \"tag\": 2}"
    else
      # Create new record (find subnet ID for this IP first)
      curl -s -X POST "${PHPIPAM_URL}/addresses/" \
        -H "Content-Type: application/json" \
        -H "token: ${TOKEN}" \
        -d "{\"subnetId\": \"5\", \"ip\": \"${IP}\", \"hostname\": \"${HOSTNAME}\", \"mac\": \"${MAC}\", \"tag\": 2}"
    fi
    ;;

  delete)
    RECORD_ID=$(curl -s "${PHPIPAM_URL}/addresses/search/${IP}/" \
      -H "token: ${TOKEN}" | jq -r '.data[0].id // empty')
    [[ -n "${RECORD_ID}" ]] && \
      curl -s -X DELETE "${PHPIPAM_URL}/addresses/${RECORD_ID}/" -H "token: ${TOKEN}"
    ;;
esac
```

Make the script executable:

```bash
sudo chmod +x /etc/dhcp/scripts/update-ipam.sh
sudo chown dhcpd:dhcpd /etc/dhcp/scripts/update-ipam.sh
```

## Approach 2: Kea DHCP with Premium Hooks

Kea DHCP (the modern replacement for ISC DHCP) has a premium hook module for direct NetBox/phpIPAM integration, but the open-source version supports run scripts:

```json
// /etc/kea/kea-dhcp4.conf
{
  "Dhcp4": {
    "hooks-libraries": [
      {
        "library": "/usr/lib/kea/hooks/libdhcp_run_script.so",
        "parameters": {
          "name": "/etc/kea/scripts/update-ipam.sh",
          "sync": false
        }
      }
    ]
  }
}
```

## Approach 3: Periodic DHCP Lease Scanning

For simpler setups, periodically parse the DHCP lease file and sync to IPAM:

```python
#!/usr/bin/env python3
# sync-dhcp-to-ipam.py — Parse ISC DHCP leases and sync to phpIPAM

import re, requests, subprocess

LEASES_FILE = "/var/lib/dhcp/dhcpd.leases"
PHPIPAM_URL = "http://phpipam.example.com/api/myapp"
TOKEN = "your-token"

# Parse active leases
lease_pattern = re.compile(
    r'lease (\d+\.\d+\.\d+\.\d+).*?'
    r'hardware ethernet ([\w:]+).*?'
    r'client-hostname "([^"]*)"',
    re.DOTALL
)

with open(LEASES_FILE) as f:
    content = f.read()

for ip, mac, hostname in lease_pattern.findall(content):
    print(f"Syncing: {ip} ({hostname} / {mac})")
    # Update phpIPAM (simplified — add error handling in production)
    requests.post(f"{PHPIPAM_URL}/addresses/",
        headers={"token": TOKEN, "Content-Type": "application/json"},
        json={"subnetId": "5", "ip": ip, "hostname": hostname, "mac": mac, "tag": 2}
    )
```

## Conclusion

Integrating your DHCP server with an IPAM tool gives you automatic, real-time IP address tracking. The hook-based approach with ISC DHCP or Kea is the most accurate — every lease event immediately updates the IPAM database, ensuring your records are always current.
