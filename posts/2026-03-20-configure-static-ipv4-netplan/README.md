# How to Configure a Static IPv4 Address with Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Netplan, Ubuntu, Static IP, IPv4, Networking

Description: Configure a static IPv4 address on Ubuntu/Debian systems using Netplan YAML configuration files, including gateway and DNS settings.

## Introduction

Netplan is the default network configuration tool on Ubuntu 17.10+. It reads YAML configuration files in `/etc/netplan/` and generates backend configurations for either NetworkManager or systemd-networkd. Static IP configuration requires setting `dhcp4: false` and specifying `addresses`, `routes`, and `nameservers`.

## Basic Static IP Configuration

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
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
# Apply the configuration
netplan apply
```

## Verify Configuration

```bash
# Check assigned IP
ip addr show eth0

# Check default route
ip route show

# Test DNS
ping -c 3 google.com
```

## Static IP with Multiple DNS and Search Domains

```yaml
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
          metric: 100
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
        search:
          - corp.example.com
          - example.com
```

## Multiple IP Addresses

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
        - 192.168.1.101/24
      routes:
        - to: default
          via: 192.168.1.1
```

## Match by MAC Address

```yaml
network:
  version: 2
  ethernets:
    myinterface:
      match:
        macaddress: "52:54:00:ab:cd:ef"
      set-name: eth0
      dhcp4: false
      addresses:
        - 10.0.0.50/24
      routes:
        - to: default
          via: 10.0.0.1
```

## Validate Before Applying

```bash
# Check YAML syntax without applying
netplan generate

# Test configuration safely (auto-reverts after 120s)
netplan try
```

## Conclusion

Static IP in Netplan uses `dhcp4: false` with `addresses`, `routes` (for the default gateway), and `nameservers`. Apply changes with `netplan apply`. Use `netplan try` for safe testing — it auto-reverts if you don't confirm within 120 seconds.
