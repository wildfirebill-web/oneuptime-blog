# How to Create a New Connection Profile with nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, nmcli, NetworkManager, Connection Profile, Networking, Configuration

Description: Create new network connection profiles using nmcli for Ethernet, WiFi, VLAN, bond, bridge, and other connection types with custom settings.

## Introduction

A NetworkManager connection profile stores all settings for a network connection — type, interface binding, IP configuration, DNS, and more. Use `nmcli connection add` to create profiles from the command line without editing configuration files directly.

## Create a Basic Ethernet Connection (DHCP)

```bash
# Create a new Ethernet connection using DHCP
nmcli connection add \
    type ethernet \
    con-name "myconnection" \
    ifname eth0

# Activate it
nmcli connection up "myconnection"
```

## Create a Static IP Connection

```bash
# Ethernet with static IP
nmcli connection add \
    type ethernet \
    con-name "static-eth0" \
    ifname eth0 \
    ipv4.method manual \
    ipv4.addresses "192.168.1.50/24" \
    ipv4.gateway "192.168.1.1" \
    ipv4.dns "8.8.8.8 1.1.1.1"

nmcli connection up "static-eth0"
```

## Create a Connection That Auto-Connects

```bash
# Auto-connect means it activates on interface appearance
nmcli connection add \
    type ethernet \
    con-name "auto-eth0" \
    ifname eth0 \
    connection.autoconnect yes \
    ipv4.method auto
```

## Create a WiFi Connection

```bash
# Create a WiFi connection profile
nmcli connection add \
    type wifi \
    con-name "home-wifi" \
    ifname wlan0 \
    ssid "MyNetworkName" \
    wifi-sec.key-mgmt wpa-psk \
    wifi-sec.psk "mypassword"

nmcli connection up "home-wifi"
```

## List All Connection Profiles

```bash
# Show all profiles
nmcli connection show

# Show only active connections
nmcli connection show --active
```

## Activate and Deactivate a Profile

```bash
# Bring up a connection
nmcli connection up "static-eth0"

# Bring down a connection
nmcli connection down "static-eth0"
```

## Delete a Connection Profile

```bash
# Remove the profile permanently
nmcli connection delete "myconnection"
```

## Clone an Existing Connection

```bash
# Duplicate a connection (useful for testing changes)
nmcli connection clone "static-eth0" "static-eth0-backup"
```

## Show Connection Profile Details

```bash
# All settings for a connection
nmcli connection show "static-eth0"

# Only IP-related settings
nmcli connection show "static-eth0" | grep ipv4
```

## Conclusion

`nmcli connection add` creates persistent connection profiles in NetworkManager. Specify `type`, `con-name`, `ifname`, and IPv4 settings in a single command. Profiles auto-connect when `connection.autoconnect yes` is set. List all profiles with `nmcli connection show` and activate with `nmcli connection up`.
