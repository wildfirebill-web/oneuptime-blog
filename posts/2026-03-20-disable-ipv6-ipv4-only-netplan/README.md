# How to Disable IPv6 and Keep IPv4 Only with Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Netplan, Ubuntu, IPv6, IPv4, Networking

Description: Disable IPv6 on Ubuntu and Debian systems using Netplan configuration, forcing interfaces to use IPv4 only.

## Introduction

Netplan disables IPv6 per-interface using `dhcp6: false` and `accept-ra: no`. For system-wide IPv6 disabling, combine Netplan settings with sysctl parameters. After applying, the interface will not configure any IPv6 addresses.

## Disable IPv6 on a Specific Interface

```yaml
# /etc/netplan/01-netcfg.yaml

network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      # Disable IPv6
      dhcp6: false
      accept-ra: no
      link-local:
        - ipv4
```

```bash
netplan apply
ip addr show eth0  # Should show no inet6 lines
```

## Disable IPv6 on DHCP Interface

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      # IPv6 disabled
      dhcp6: false
      accept-ra: no
```

## Disable IPv6 on All Interfaces

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: false
      accept-ra: no
    eth1:
      dhcp4: false
      addresses:
        - 10.0.0.10/24
      dhcp6: false
      accept-ra: no
```

## Disable IPv6 System-Wide with sysctl

Combine Netplan settings with sysctl for a complete disable:

```bash
# Create sysctl configuration
cat > /etc/sysctl.d/99-disable-ipv6.conf << 'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF

# Apply immediately
sysctl -p /etc/sysctl.d/99-disable-ipv6.conf
```

## Disable IPv6 via Kernel Boot Parameter

For systems where IPv6 must be disabled before network comes up:

```bash
# Add to GRUB_CMDLINE_LINUX in /etc/default/grub
GRUB_CMDLINE_LINUX="ipv6.disable=1"

# Update GRUB
update-grub
# Reboot required
```

## Verify IPv6 is Disabled

```bash
# Check interface has no IPv6 address
ip -6 addr show eth0
# Should show: no output or only link-local fe80:: if accept-ra still yes

# Check disable_ipv6 sysctl
cat /proc/sys/net/ipv6/conf/eth0/disable_ipv6
# 1 = disabled
```

## Conclusion

Disable IPv6 in Netplan by setting `dhcp6: false`, `accept-ra: no`, and `link-local: [ipv4]` per interface. For system-wide IPv6 disable, also add `net.ipv6.conf.all.disable_ipv6=1` to sysctl. Apply Netplan changes with `netplan apply`.
