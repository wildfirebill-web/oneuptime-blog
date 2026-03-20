# How to Configure a VLAN with nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, nmcli, NetworkManager, VLAN, 802.1Q, Networking

Description: Configure 802.1Q VLAN subinterfaces on Linux using nmcli, including VLAN creation, IP assignment, and persistent configuration.

## Introduction

NetworkManager supports VLAN subinterfaces natively. Use `nmcli connection add type vlan` to create a VLAN connection that associates a VLAN ID with a parent interface. The configuration persists across reboots.

## Create a VLAN Connection

```bash
# Create VLAN 10 on eth0 with a static IP
nmcli connection add \
    type vlan \
    con-name "vlan10" \
    dev eth0 \
    id 10 \
    ipv4.method manual \
    ipv4.addresses "192.168.10.1/24" \
    ipv4.gateway "192.168.10.254"

# Activate
nmcli connection up "vlan10"
```

## Create a VLAN with DHCP

```bash
nmcli connection add \
    type vlan \
    con-name "vlan20" \
    dev eth0 \
    id 20 \
    ipv4.method auto

nmcli connection up "vlan20"
```

## Verify the VLAN Interface

```bash
# Check the VLAN interface was created
ip link show eth0.10

# Verify the IP address
ip addr show eth0.10

# Show connection status
nmcli connection show "vlan10"
```

## Create Multiple VLANs on One Interface

```bash
# VLAN 10
nmcli connection add type vlan con-name "vlan10" dev eth0 id 10 \
    ipv4.method manual ipv4.addresses "192.168.10.1/24"

# VLAN 20
nmcli connection add type vlan con-name "vlan20" dev eth0 id 20 \
    ipv4.method manual ipv4.addresses "192.168.20.1/24"

# VLAN 30
nmcli connection add type vlan con-name "vlan30" dev eth0 id 30 \
    ipv4.method auto

nmcli connection up "vlan10"
nmcli connection up "vlan20"
nmcli connection up "vlan30"
```

## Delete a VLAN Connection

```bash
# Bring down and delete the VLAN connection
nmcli connection down "vlan10"
nmcli connection delete "vlan10"
```

## Show VLAN Details

```bash
# Show all VLAN connections
nmcli connection show | grep vlan

# Show detailed VLAN settings
nmcli connection show "vlan10" | grep -i vlan
```

## Load 8021q Module (if VLANs not working)

```bash
# Ensure the 8021q kernel module is loaded
modprobe 8021q

# Make it persistent
echo "8021q" >> /etc/modules
```

## Conclusion

VLAN subinterfaces with nmcli use `nmcli connection add type vlan dev <parent> id <vlan-id>`. NetworkManager automatically creates the `eth0.10`-style interface and manages the VLAN lifetime. Activate with `nmcli connection up` and verify with `ip link show`.
