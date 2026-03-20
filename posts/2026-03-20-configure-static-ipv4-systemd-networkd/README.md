# How to Configure a Static IPv4 Address with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, Static IP, IPv4, Networking, Configuration

Description: Configure a static IPv4 address on a Linux interface using systemd-networkd .network files, including gateway and DNS settings.

## Introduction

systemd-networkd configures network interfaces through `.network` files in `/etc/systemd/network/`. A static IPv4 configuration specifies the address, prefix length, gateway, and optionally DNS in the file — no DHCP required.

## Create the .network File

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=8.8.4.4
```

```bash
# Apply the configuration
systemctl restart systemd-networkd
```

## Verify the Configuration

```bash
# Check assigned IP address
ip addr show eth0

# Check default route
ip route show

# Check DNS (if using systemd-resolved)
resolvectl status eth0
```

## Multiple IP Addresses on One Interface

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Address=192.168.1.101/24
Gateway=192.168.1.1
DNS=8.8.8.8
```

## Static IP with Custom MTU

```ini
[Match]
Name=eth0

[Link]
MTUBytes=9000

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
```

## Match by MAC Address Instead of Interface Name

```ini
# Useful when interface names change between reboots
[Match]
MACAddress=52:54:00:ab:cd:ef

[Network]
Address=10.0.0.50/24
Gateway=10.0.0.1
```

## Reload Configuration Without Restart

```bash
# Reload .network files without restarting the daemon
networkctl reload

# Force reconfiguration of eth0 specifically
networkctl reconfigure eth0
```

## Check Interface Status

```bash
# Shows IP, DNS, gateway, and operational state
networkctl status eth0
```

## File Naming and Priority

Files in `/etc/systemd/network/` are processed in lexicographic order. Prefix with numbers to control priority:

```
/etc/systemd/network/
  10-eth0.network   ← applied first (lower number)
  20-eth1.network
  30-wlan0.network
```

## Conclusion

Static IPv4 configuration with systemd-networkd requires creating a `.network` file with `[Match]` (to identify the interface) and `[Network]` (for IP, gateway, DNS) sections. Apply with `networkctl reload` or `systemctl restart systemd-networkd`. Use `networkctl status eth0` to verify the configuration is active.
