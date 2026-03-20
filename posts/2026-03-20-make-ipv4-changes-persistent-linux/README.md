# How to Make IPv4 Address Changes Persistent Across Reboots on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, IPv4, Netplan, NetworkManager, Persistent Configuration

Description: Make IPv4 address, gateway, and DNS settings persistent across reboots on Linux using Netplan, /etc/network/interfaces, NetworkManager, and systemd-networkd.

## Introduction

Changes made with `ip addr`, `ip route`, and `dhclient` are ephemeral — they disappear when the system reboots. Persistence requires writing the configuration to a file that the network management subsystem reads at boot.

## Method 1: Netplan (Ubuntu 18.04+)

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
# Test without committing (auto-reverts if you do not confirm within 120s)
sudo netplan try
```

## Method 2: /etc/network/interfaces (Debian/Ubuntu)

```
# /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

```bash
sudo systemctl restart networking
# Or for single interface
sudo ifdown eth0 && sudo ifup eth0
```

## Method 3: NetworkManager with nmcli

```bash
# Modify the existing connection
nmcli con mod "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses "192.168.1.100/24" \
  ipv4.gateway "192.168.1.1" \
  ipv4.dns "8.8.8.8,1.1.1.1"

# Activate the change
nmcli con up "Wired connection 1"
```

The connection is stored in `/etc/NetworkManager/system-connections/`.

## Method 4: systemd-networkd

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=1.1.1.1
```

```bash
sudo systemctl enable --now systemd-networkd
sudo networkctl reload
```

## Method 5: RHEL/CentOS /etc/sysconfig/network-scripts/

```bash
# /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.1.100
PREFIX=24
GATEWAY=192.168.1.1
DNS1=8.8.8.8
```

```bash
sudo nmcli con reload
sudo nmcli con up eth0
```

## Verifying Persistence After Reboot

```bash
# Reboot and verify
sudo reboot
# After login:
ip -4 addr show eth0
ip route show default
```

## Conclusion

Choose the persistence method that matches your distribution and network manager: Netplan for modern Ubuntu, `/etc/network/interfaces` for Debian/legacy Ubuntu, NetworkManager via `nmcli` for desktop systems, and systemd-networkd for server deployments. Always test with `netplan try` or a controlled reboot after making changes.
