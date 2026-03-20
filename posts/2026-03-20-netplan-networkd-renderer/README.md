# How to Use Netplan with the networkd Renderer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Netplan, systemd-networkd, Linux, Networking, Ubuntu, Server

Description: Configure Netplan to use the systemd-networkd renderer for server-grade network management on Ubuntu headless and cloud instances.

## Introduction

Netplan supports two backend renderers: `networkd` (systemd-networkd) and `NetworkManager`. The `networkd` renderer is the preferred choice for servers and cloud instances because it is lightweight, starts early in the boot process, and has no GUI dependencies.

## When to Use networkd

- Headless servers and cloud VMs
- Containers (LXC/LXD)
- Systems without a desktop environment
- Scenarios requiring fast, predictable network bring-up during boot

## Basic Configuration

Set `renderer: networkd` in your Netplan YAML:

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd          # Use systemd-networkd as the backend
  ethernets:
    eth0:
      dhcp4: true             # Obtain IPv4 address via DHCP
      dhcp6: false            # Disable IPv6 DHCP
```

For a static IP configuration:

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 10.0.0.5/24
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 9.9.9.9
```

## Enabling and Verifying systemd-networkd

Make sure `systemd-networkd` is enabled and running:

```bash
# Enable and start systemd-networkd
sudo systemctl enable --now systemd-networkd

# Check its status
systemctl status systemd-networkd
```

Apply the Netplan configuration:

```bash
sudo netplan apply
```

## Inspecting networkd State

Once active, use `networkctl` to inspect interface state managed by networkd:

```bash
# List all managed interfaces and their status
networkctl list

# Show detailed info for a specific interface
networkctl status eth0
```

## Configuring a Bond with networkd

The `networkd` renderer excels at bond and bridge configurations:

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  bonds:
    bond0:
      interfaces: [eth0, eth1]      # Aggregate two NICs
      parameters:
        mode: active-backup         # Failover mode
        primary: eth0
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
```

## Conclusion

The `networkd` renderer is the right choice for most server deployments. It is stable, well-integrated with systemd, and supports all the networking primitives needed for production infrastructure.
