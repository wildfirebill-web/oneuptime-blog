# How to Create a .network File for systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, .network file, Configuration, Networking

Description: Create and configure .network files for systemd-networkd to manage interface addressing, routing, and DNS declaratively.

## Introduction

A `.network` file tells systemd-networkd how to configure a matched network interface. Files live in `/etc/systemd/network/` and are named with numeric prefixes to control processing order. Each file contains sections like `[Match]`, `[Network]`, `[Address]`, `[Route]`, and `[DHCPv4]`.

## Basic .network File Structure

```ini
# /etc/systemd/network/10-eth0.network

# [Match] selects which interface(s) this file applies to
[Match]
Name=eth0

# [Network] defines addressing and protocol settings
[Network]
Address=10.0.0.10/24
Gateway=10.0.0.1
DNS=8.8.8.8
```

## Match Section Options

```ini
[Match]
# Match by interface name (supports globs: eth*, en*)
Name=eth0

# Match by MAC address
MACAddress=52:54:00:ab:cd:ef

# Match by type
Type=ether

# Match by driver
Driver=e1000e
```

## Network Section Options

```ini
[Network]
# Static IP
Address=192.168.1.50/24
Gateway=192.168.1.1
DNS=1.1.1.1

# Or use DHCP
DHCP=ipv4

# Link local addressing
LinkLocalAddressing=no

# IPv6 settings
IPv6AcceptRA=no
```

## Separate [Address] Section

```ini
# Multiple addresses using [Address] sections
[Match]
Name=eth0

[Network]
Gateway=192.168.1.1

[Address]
Address=192.168.1.10/24

[Address]
Address=192.168.1.11/24
Label=eth0:1
```

## Static Route in .network File

```ini
[Match]
Name=eth0

[Network]
Address=10.0.0.10/24
Gateway=10.0.0.1

[Route]
Destination=172.16.0.0/24
Gateway=10.0.0.2
Metric=100
```

## VLAN Configuration in .network File

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
# Attach VLANs to this interface
VLAN=eth0.10
VLAN=eth0.20
```

## Apply and Verify

```bash
# Reload .network files
networkctl reload

# Check interface status
networkctl status eth0

# Show assigned addresses
ip addr show eth0
```

## File Naming Convention

```
/etc/systemd/network/
  10-lo.network       ← loopback (low number = high priority)
  20-eth0.network     ← primary interface
  30-eth1.network     ← secondary interface
  40-wlan0.network    ← wireless
```

Files are processed alphabetically — use numeric prefixes to control order.

## Conclusion

`.network` files are the core configuration unit for systemd-networkd. Use `[Match]` to select interfaces and `[Network]` to configure IP addressing, gateways, DNS, and protocol settings. Additional sections like `[Route]`, `[Address]`, and `[DHCPv4]` provide fine-grained control. Apply changes with `networkctl reload`.
