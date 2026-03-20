# How to Configure IPv4 Addresses on macOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: macOS, IPv4, Networking, Network Configuration, Static IP, sysadmin

Description: IPv4 addresses on macOS can be configured through System Settings, the networksetup command-line tool, or temporary assignments with the ifconfig command.

## Method 1: GUI (System Settings)

1. Open **System Settings → Network**.
2. Select your network interface (Wi-Fi or Ethernet).
3. Click **Details** (macOS Ventura+) or the gear icon.
4. Go to the **TCP/IP** tab.
5. Change **Configure IPv4** from "Using DHCP" to **Manually**.
6. Enter:
   - IP Address: `192.168.1.100`
   - Subnet Mask: `255.255.255.0`
   - Router (Gateway): `192.168.1.1`
7. Click **OK** and **Apply**.

## Method 2: networksetup (Command Line)

`networksetup` is the macOS CLI for network configuration and persists across reboots:

```bash
# List all network services
networksetup -listallnetworkservices

# Assign a static IP to "Wi-Fi" (replace with your service name)
sudo networksetup -setmanual "Wi-Fi" 192.168.1.100 255.255.255.0 192.168.1.1

# Set DNS servers
sudo networksetup -setdnsservers "Wi-Fi" 8.8.8.8 1.1.1.1

# Verify
networksetup -getinfo "Wi-Fi"
networksetup -getdnsservers "Wi-Fi"

# Revert to DHCP
sudo networksetup -setdhcp "Wi-Fi"
```

## Method 3: ifconfig (Temporary)

Changes with `ifconfig` are lost after reboot:

```bash
# Assign a static IP temporarily (en0 is typically Wi-Fi on macOS)
sudo ifconfig en0 192.168.1.100 netmask 255.255.255.0

# Add a default route
sudo route add default 192.168.1.1

# View current configuration
ifconfig en0
```

## Adding an IP Alias

```bash
# Add a secondary IP address to en0
sudo ifconfig en0 alias 192.168.1.200 255.255.255.0

# Remove the alias
sudo ifconfig en0 -alias 192.168.1.200
```

## Flushing the DNS Cache

After changing DNS settings, flush the cache:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

## Verifying the Configuration

```bash
# Full interface info
ifconfig en0

# Routing table
netstat -nr

# DNS resolution test
nslookup google.com

# Connectivity test
ping -c 4 8.8.8.8
```

## Key Takeaways

- `networksetup` is the recommended CLI tool for persistent macOS network configuration.
- `ifconfig` changes are temporary; use `networksetup` for persistence.
- Service names (like "Wi-Fi", "Ethernet") may differ; use `networksetup -listallnetworkservices` to find yours.
- Flush the DNS cache after changing DNS servers with `dscacheutil -flushcache`.
