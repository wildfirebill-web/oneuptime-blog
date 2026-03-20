# How to Configure Dual-Stack IPv4/IPv6 Networking on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dual-Stack, IPv4, IPv6, Linux, Networking, Configuration

Description: Configure a Linux server to run dual-stack networking with both IPv4 and IPv6 addresses on the same interface, verify connectivity, and manage routing for both protocols.

## Introduction

Dual-stack networking allows a host to communicate over both IPv4 and IPv6 simultaneously. Both protocol families are active on the same interface, and applications can accept connections on either. This is the recommended transition strategy for IPv6 adoption.

## Checking Current Network Configuration

```bash
# View current interface addresses (both IPv4 and IPv6)

ip address show

# Check IPv4 addresses
ip -4 address show

# Check IPv6 addresses
ip -6 address show

# View routing tables
ip -4 route show
ip -6 route show
```

## Static Dual-Stack Configuration (netplan)

```yaml
# /etc/netplan/01-dual-stack.yaml (Ubuntu 18.04+)

network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 10.0.0.5/24          # IPv4
        - 2001:db8::5/64       # IPv6
      routes:
        - to: default
          via: 10.0.0.1        # IPv4 default gateway
        - to: ::/0
          via: 2001:db8::1     # IPv6 default gateway
      nameservers:
        addresses:
          - 8.8.8.8            # IPv4 DNS
          - 2001:4860:4860::8888  # IPv6 DNS
```

```bash
sudo netplan apply
sudo netplan try   # Test before applying (auto-reverts on failure)
```

## Static Dual-Stack (interfaces file)

```bash
# /etc/network/interfaces (Debian)

auto eth0
iface eth0 inet static
    address 10.0.0.5
    netmask 255.255.255.0
    gateway 10.0.0.1

iface eth0 inet6 static
    address 2001:db8::5
    netmask 64
    gateway 2001:db8::1
```

## Enabling IPv6 (if disabled)

```bash
# Check if IPv6 is disabled
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
# 1 = disabled, 0 = enabled

# Enable IPv6
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=0

# Make persistent
echo "net.ipv6.conf.all.disable_ipv6=0" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6=0" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Verifying Dual-Stack Connectivity

```bash
# Test IPv4 connectivity
ping -4 -c 3 8.8.8.8

# Test IPv6 connectivity
ping -6 -c 3 2001:4860:4860::8888

# Test DNS resolution for both
host google.com     # Shows A and AAAA records

# Verify both addresses are assigned
ip addr show eth0 | grep -E "inet |inet6"

# Check which protocol curl uses (prefers IPv6 by default)
curl -v -4 https://example.com   # Force IPv4
curl -v -6 https://example.com   # Force IPv6
```

## Conclusion

Dual-stack networking is enabled by adding both IPv4 (`inet`) and IPv6 (`inet6`) address families to the same interface. Use netplan, `interfaces`, or `NetworkManager` depending on your distribution. Verify connectivity with `ping -4` and `ping -6`. Most modern applications automatically use both protocols and prefer IPv6 when available. The dual-stack approach requires no special transition mechanisms-both protocols coexist natively.
