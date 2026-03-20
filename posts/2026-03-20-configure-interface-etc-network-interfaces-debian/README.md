# How to Configure a Network Interface Using /etc/network/interfaces on Debian

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Debian, Networking, IPv4, Network Configuration, Ifupdown

Description: Configure static and DHCP IPv4 addresses, routes, and DNS on Debian using /etc/network/interfaces and the ifupdown system.

## Introduction

Debian and older Ubuntu systems use `/etc/network/interfaces` with the `ifupdown` toolset (`ifup`/`ifdown`) to manage network configuration. This file-based approach is simple, predictable, and does not require a running daemon.

## Basic DHCP Configuration

```text
# /etc/network/interfaces

# Loopback

auto lo
iface lo inet loopback

# DHCP on eth0 - request IP from DHCP server automatically at boot
auto eth0
iface eth0 inet dhcp
```

The `auto eth0` directive brings the interface up at boot. Without it, you must run `sudo ifup eth0` manually.

## Static IP Configuration

```text
# /etc/network/interfaces

auto lo
iface lo inet loopback

# Static IPv4 on eth0
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
    dns-search example.com
```

## Multiple Addresses (Secondary)

```text
# Primary interface
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1

# Secondary alias
auto eth0:1
iface eth0:1 inet static
    address 192.168.1.101
    netmask 255.255.255.0
```

## Static Routes with post-up

```text
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    # Add route after interface comes up
    post-up ip route add 10.0.0.0/8 via 192.168.1.254
    # Remove route before interface goes down
    pre-down ip route del 10.0.0.0/8 via 192.168.1.254
```

## Running Commands After Interface Up

```text
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    post-up ethtool -s eth0 speed 1000 duplex full
    post-up arp -s 192.168.1.200 00:aa:bb:cc:dd:ee
```

## Applying Changes

```bash
# Apply changes to eth0 without rebooting
sudo ifdown eth0 && sudo ifup eth0

# Or restart the entire networking service
sudo systemctl restart networking

# Verify
ip -4 addr show eth0
```

## Including Additional Files

For large configurations, split into directory:

```text
# At the end of /etc/network/interfaces:
source /etc/network/interfaces.d/*.cfg
```

Then create per-interface files in `/etc/network/interfaces.d/`:

```bash
sudo tee /etc/network/interfaces.d/eth0.cfg << 'EOF'
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
EOF
```

## Conclusion

`/etc/network/interfaces` provides a clean, daemon-free way to configure networking on Debian. Use `auto` for interfaces that should come up at boot, `post-up` for custom commands, and `source` to keep large configurations organized. Restart with `ifdown`/`ifup` for individual interfaces or `systemctl restart networking` for all.
