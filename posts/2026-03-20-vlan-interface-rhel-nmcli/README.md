# How to Create a VLAN Interface on RHEL Using nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RHEL, VLAN, nmcli, NetworkManager, 802.1Q, Networking

Description: Create and configure a persistent 802.1Q VLAN interface on Red Hat Enterprise Linux using nmcli and NetworkManager.

## Introduction

On RHEL, CentOS, and Fedora, NetworkManager manages network configuration and `nmcli` is the command-line interface. nmcli makes it easy to create VLAN connections that are persistent by default - no separate save step is required.

## Prerequisites

- RHEL 7/8/9 or compatible (CentOS, Fedora)
- NetworkManager running (`systemctl status NetworkManager`)
- Root or sudo access

## Create a VLAN Connection

```bash
# Create VLAN 100 on top of eth0 with a static IP

nmcli connection add \
    type vlan \
    con-name "eth0.100" \
    dev eth0 \
    id 100 \
    ipv4.addresses "192.168.100.1/24" \
    ipv4.method manual

# Activate the connection
nmcli connection up "eth0.100"
```

## Verify the VLAN Interface

```bash
# Show the connection details
nmcli connection show "eth0.100"

# Show interface IP
ip addr show eth0.100

# Check VLAN-specific details
ip -d link show eth0.100
```

## Create a VLAN with DHCP

```bash
nmcli connection add \
    type vlan \
    con-name "vlan-dhcp" \
    dev eth0 \
    id 200 \
    ipv4.method auto
```

## Add DNS and Gateway

```bash
# Add gateway and DNS to an existing VLAN connection
nmcli connection modify "eth0.100" \
    ipv4.gateway "192.168.100.254" \
    ipv4.dns "8.8.8.8 8.8.4.4"

# Apply changes
nmcli connection up "eth0.100"
```

## Create Multiple VLANs

```bash
# Management VLAN (VLAN 10)
nmcli connection add type vlan con-name vlan10 dev eth0 id 10 \
    ipv4.addresses 10.10.0.1/24 ipv4.method manual

# Production VLAN (VLAN 20)
nmcli connection add type vlan con-name vlan20 dev eth0 id 20 \
    ipv4.addresses 10.20.0.1/24 ipv4.method manual

# Activate both
nmcli connection up vlan10
nmcli connection up vlan20
```

## Using a Custom Interface Name

```bash
# Force interface name using ifname
nmcli connection add \
    type vlan \
    con-name "management" \
    ifname "mgmt" \
    dev eth0 \
    id 100 \
    ipv4.addresses "172.16.0.1/24" \
    ipv4.method manual
```

## List, Deactivate, and Delete

```bash
# List all VLAN connections
nmcli connection show | grep vlan

# Deactivate (bring down) a VLAN
nmcli connection down "eth0.100"

# Permanently delete a VLAN connection
nmcli connection delete "eth0.100"
```

## Conclusion

nmcli on RHEL creates persistent VLAN connections through NetworkManager. The `type vlan` combined with `dev` (parent interface) and `id` (VLAN ID) is all that is needed. Connections are automatically active after reboot. Use `nmcli connection modify` to update settings and `nmcli connection up` to apply changes immediately.
