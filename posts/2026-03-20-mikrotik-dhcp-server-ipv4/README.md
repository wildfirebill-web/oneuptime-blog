# How to Set Up DHCP Server for IPv4 on MikroTik

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MikroTik, RouterOS, DHCP, IPv4, Network Configuration

Description: Configure a DHCP server on MikroTik RouterOS to automatically assign IPv4 addresses to clients, set static leases for specific devices, and configure DHCP relay for multiple subnets.

## Introduction

MikroTik's built-in DHCP server handles IPv4 address assignment for LAN clients. Configuration involves creating a DHCP pool, a server definition on the interface, and optional static mappings for devices requiring fixed IPs.

## Quick Setup with DHCP Setup Wizard

```mikrotik
# Automated wizard — follow the prompts
/ip dhcp-server setup

# Select interface: ether2 (LAN)
# Address space: 192.168.1.0/24
# Gateway: 192.168.1.1
# Pool: 192.168.1.100-192.168.1.200
# DNS servers: 8.8.8.8,8.8.4.4
# Lease time: 10m (or as desired)
```

## Manual DHCP Server Configuration

```mikrotik
# Step 1: Create address pool
/ip pool add name=LAN-POOL ranges=192.168.1.100-192.168.1.200

# Step 2: Create DHCP server
/ip dhcp-server add \
  name=LAN-DHCP \
  interface=ether2 \
  address-pool=LAN-POOL \
  lease-time=12h \
  disabled=no

# Step 3: Create network definition (gateway, DNS, domain)
/ip dhcp-server network add \
  address=192.168.1.0/24 \
  gateway=192.168.1.1 \
  dns-server=8.8.8.8,8.8.4.4 \
  domain=local.lan \
  comment="LAN Network"
```

## Static DHCP Lease (Fixed IP by MAC)

```mikrotik
/ip dhcp-server lease add \
  address=192.168.1.10 \
  mac-address=00:1A:2B:3C:4D:5E \
  server=LAN-DHCP \
  comment="Office-Printer"

/ip dhcp-server lease add \
  address=192.168.1.11 \
  mac-address=AA:BB:CC:DD:EE:FF \
  server=LAN-DHCP \
  comment="FileServer"
```

## Convert Dynamic Lease to Static

```mikrotik
# Show current dynamic leases
/ip dhcp-server lease print

# Make a dynamic lease permanent
/ip dhcp-server lease make-static 0
```

## DHCP Relay for Multiple Subnets

```mikrotik
# When DHCP server is on different router than clients
/ip dhcp-relay add \
  name=RELAY-VLAN10 \
  interface=vlan10 \
  dhcp-server=10.1.1.20 \
  local-address=10.1.10.1 \
  disabled=no
```

## View Leases

```mikrotik
# Show active leases
/ip dhcp-server lease print

# Show with detail (expiry time, hostname)
/ip dhcp-server lease print detail

# Show only active (not expired)
/ip dhcp-server lease print where status=bound
```

## DHCP Options (Option 150 for VoIP)

```mikrotik
/ip dhcp-server option add \
  name=cisco-tftp \
  code=150 \
  value="'10.1.30.5'"

/ip dhcp-server option sets add name=VOIP-OPTIONS options=cisco-tftp

/ip dhcp-server network set 0 dhcp-option-set=VOIP-OPTIONS
```

## Conclusion

MikroTik DHCP server configuration takes three steps: create a pool, add a server on the interface, and define network parameters. Use static leases for servers and printers requiring fixed IPs, DHCP relay to serve multiple VLANs from a central server, and DHCP options for VoIP phone provisioning.
