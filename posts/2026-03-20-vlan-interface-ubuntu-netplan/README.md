# How to Create a VLAN Interface on Ubuntu Using Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ubuntu, VLAN, Netplan, 802.1Q, Networking, Network Configuration

Description: Configure a persistent 802.1Q VLAN subinterface on Ubuntu using Netplan YAML configuration files with the vlans key.

## Introduction

Ubuntu uses Netplan as its default network configuration tool. Netplan supports VLAN configuration through a dedicated `vlans` key in YAML configuration files. Changes are persistent and applied cleanly through Netplan's backend (either NetworkManager or systemd-networkd).

## Prerequisites

- Ubuntu 18.04 or later
- Netplan installed (default on Ubuntu)
- The `8021q` kernel module (auto-loaded by Netplan)
- Root or sudo access

## Basic VLAN Configuration

Netplan configuration files live in `/etc/netplan/`. Create or edit a file to define the VLAN:

```yaml
# /etc/netplan/01-vlan.yaml

network:
  version: 2
  renderer: networkd  # or: renderer: NetworkManager

  ethernets:
    eth0:
      # Parent interface must be defined but doesn't need an IP
      dhcp4: false

  vlans:
    eth0.100:
      id: 100
      link: eth0
      addresses:
        - 192.168.100.1/24
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

## Apply the Configuration

```bash
# Validate the YAML syntax first
netplan generate

# Apply the configuration (disrupts networking briefly)
netplan apply

# Verify the VLAN interface
ip addr show eth0.100
```

## Multiple VLANs on One Interface

```yaml
# /etc/netplan/01-vlans.yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false

  vlans:
    eth0.10:
      id: 10
      link: eth0
      addresses:
        - 10.10.0.1/24

    eth0.20:
      id: 20
      link: eth0
      addresses:
        - 10.20.0.1/24

    eth0.30:
      id: 30
      link: eth0
      dhcp4: true
```

## VLAN with Static Routes

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false

  vlans:
    eth0.100:
      id: 100
      link: eth0
      addresses:
        - 10.100.0.1/24
      routes:
        - to: 10.200.0.0/24
          via: 10.100.0.254
```

## Using a Custom VLAN Interface Name

```yaml
vlans:
  management:        # Custom name instead of eth0.100
    id: 100
    link: eth0
    addresses:
      - 172.16.0.1/24
```

## Verify After Applying

```bash
# Check VLAN interface is up with correct IP
ip addr show eth0.100

# Show VLAN details
ip -d link show eth0.100

# Check routes
ip route show dev eth0.100

# Test VLAN connectivity
ping -I eth0.100 -c 3 192.168.100.254
```

## Troubleshoot Common Issues

```bash
# Check for YAML syntax errors
netplan generate

# Check networkd/NetworkManager applied the config
networkctl status eth0.100

# Check 8021q module is loaded
lsmod | grep 8021q
```

## Conclusion

Netplan makes VLAN configuration on Ubuntu straightforward. Define the parent interface in `ethernets`, then declare VLANs in the `vlans` section with the `id` and `link` keys. Apply changes with `netplan apply` and verify with `ip addr show`. Configuration is persistent across reboots automatically.
