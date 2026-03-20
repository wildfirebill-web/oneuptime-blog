# How to Configure a VLAN with Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Netplan, Ubuntu, VLAN, 802.1Q, Networking

Description: Configure 802.1Q VLAN subinterfaces on Ubuntu and Debian using Netplan YAML, including static IP and DHCP on VLAN interfaces.

## Introduction

Netplan supports VLAN configuration under the `vlans` key. Each VLAN entry specifies the VLAN ID and the parent (link) interface. VLAN interfaces are automatically created when `netplan apply` runs.

## Basic VLAN Configuration

```yaml
# /etc/netplan/01-netcfg.yaml

network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false

  vlans:
    eth0.10:
      id: 10
      link: eth0
      dhcp4: false
      addresses:
        - 192.168.10.1/24
      routes:
        - to: default
          via: 192.168.10.254
```

```bash
netplan apply
ip link show eth0.10
ip addr show eth0.10
```

## VLAN with DHCP

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false

  vlans:
    eth0.20:
      id: 20
      link: eth0
      dhcp4: true
```

## Multiple VLANs on One Interface

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false

  vlans:
    eth0.10:
      id: 10
      link: eth0
      dhcp4: false
      addresses:
        - 192.168.10.1/24

    eth0.20:
      id: 20
      link: eth0
      dhcp4: false
      addresses:
        - 192.168.20.1/24

    eth0.30:
      id: 30
      link: eth0
      dhcp4: true
```

## VLAN with DNS

```yaml
vlans:
  eth0.10:
    id: 10
    link: eth0
    dhcp4: false
    addresses:
      - 192.168.10.1/24
    nameservers:
      addresses:
        - 192.168.10.254
        - 8.8.8.8
      search:
        - vlan10.example.com
```

## Apply and Verify

```bash
# Apply
netplan apply

# Check VLAN interface
ip -d link show eth0.10

# Check IP on VLAN
ip addr show eth0.10

# Test connectivity
ping -c 3 192.168.10.254
```

## Load 8021q Module (if needed)

```bash
# Load VLAN kernel module
modprobe 8021q

# Make persistent
echo "8021q" >> /etc/modules
```

## Conclusion

Netplan VLAN configuration uses the `vlans` section with `id` (VLAN ID) and `link` (parent interface) for each VLAN. Define IP settings just like Ethernet interfaces. Parent interface should have `dhcp4: false` with no IP of its own when used purely as a VLAN trunk. Apply with `netplan apply`.
