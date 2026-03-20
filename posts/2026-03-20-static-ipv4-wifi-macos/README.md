# How to Configure a Static IPv4 Address for WiFi on macOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WiFi, macOS, Static IP, IPv4, Network Configuration

Description: Learn how to configure a static IPv4 address for WiFi on macOS using System Preferences and the networksetup command-line tool.

## When to Use a Static WiFi IP on macOS

Static IPs on macOS WiFi are useful for:
- Configuring router port forwarding to a Mac
- Running local development servers accessible at a fixed IP
- Network labs and testing environments
- When the local DHCP server is unreliable

## Step 1: Check Current Network Configuration

```bash
# View current IP, gateway, and DNS via command line

ifconfig en0
# en0 is typically the WiFi adapter on Macs

# Or with networksetup
networksetup -getinfo "Wi-Fi"
# Output:
# DHCP Configuration
# IP address: 192.168.1.105
# Subnet mask: 255.255.255.0
# Router: 192.168.1.1

# Check interface name
networksetup -listallnetworkservices
```

## Step 2: Configure Static IP via System Preferences

1. Open **System Preferences** (or **System Settings** on macOS Ventura+)
2. Click **Network**
3. Select **Wi-Fi** in the left panel
4. Click **Advanced** (bottom right)
5. Go to **TCP/IP** tab
6. Change **Configure IPv4** from **Using DHCP** to **Manually**
7. Enter:
   - **IPv4 Address**: 192.168.1.50
   - **Subnet Mask**: 255.255.255.0
   - **Router**: 192.168.1.1
8. Go to **DNS** tab, click **+**, add `8.8.8.8` and `8.8.4.4`
9. Click **OK** → **Apply**

## Step 3: Configure Static IP via networksetup

```bash
# Set static IP address (requires sudo)
sudo networksetup -setmanual "Wi-Fi" 192.168.1.50 255.255.255.0 192.168.1.1

# Set DNS servers
sudo networksetup -setdnsservers "Wi-Fi" 8.8.8.8 8.8.4.4

# Verify the configuration
networksetup -getinfo "Wi-Fi"
# Output:
# Manual Configuration
# IP address: 192.168.1.50
# Subnet mask: 255.255.255.0
# Router: 192.168.1.1

networksetup -getdnsservers "Wi-Fi"
# 8.8.8.8
# 8.8.4.4
```

## Step 4: Test the Static IP Configuration

```bash
# Test local gateway
ping -c 3 192.168.1.1

# Test internet connectivity
ping -c 3 8.8.8.8

# Test DNS resolution
nslookup google.com

# Check full interface info
ifconfig en0
```

## Step 5: Configure Multiple Locations

macOS supports named network "Locations" for switching between configurations:

```bash
# Create a new location for static IP
# System Preferences → Network → Location → Edit Locations → +
# Name: "StaticHome"
# Configure as needed

# Switch locations from command line
sudo scselect "StaticHome"

# Switch back to automatic
sudo scselect "Automatic"

# List available locations
networksetup -listlocations
```

## Step 6: Revert to DHCP

```bash
# Set back to DHCP
sudo networksetup -setdhcp "Wi-Fi"

# Clear manual DNS settings
sudo networksetup -setdnsservers "Wi-Fi" empty

# Renew DHCP lease
sudo ipconfig set en0 DHCP
# Or disconnect and reconnect WiFi

# Verify
networksetup -getinfo "Wi-Fi"
# Should show: DHCP Configuration
```

## Step 7: Automate with a Shell Script

```bash
#!/bin/bash
# Toggle between static and DHCP on macOS WiFi

IFACE="Wi-Fi"
STATIC_IP="192.168.1.50"
MASK="255.255.255.0"
GATEWAY="192.168.1.1"

current=$(networksetup -getinfo "$IFACE" | head -1)

if [[ "$current" == "DHCP Configuration" ]]; then
  echo "Setting static IP $STATIC_IP..."
  sudo networksetup -setmanual "$IFACE" $STATIC_IP $MASK $GATEWAY
  sudo networksetup -setdnsservers "$IFACE" 8.8.8.8 8.8.4.4
else
  echo "Setting DHCP..."
  sudo networksetup -setdhcp "$IFACE"
  sudo networksetup -setdnsservers "$IFACE" empty
fi

networksetup -getinfo "$IFACE"
```

## Conclusion

Configure a static WiFi IP on macOS via System Preferences → Network → Wi-Fi → Advanced → TCP/IP, or via the command line with `sudo networksetup -setmanual "Wi-Fi" <IP> <mask> <gateway>`. Use macOS Locations to quickly switch between static and DHCP configurations. Always verify with `ping` to the gateway and `nslookup` after configuration, and revert to DHCP with `sudo networksetup -setdhcp "Wi-Fi"` when static IP is no longer needed.
