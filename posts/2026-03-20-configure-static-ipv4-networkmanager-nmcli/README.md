# How to Configure a Static IPv4 Address with NetworkManager and nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, NetworkManager, nmcli, IPv4, Network Configuration, RHEL

Description: Configure a static IPv4 address using nmcli to create or modify a NetworkManager connection profile, set the gateway and DNS, and activate the configuration.

## Introduction

NetworkManager is the default network management daemon on RHEL, Fedora, CentOS, and many desktop Linux distributions. `nmcli` is its command-line interface, enabling scriptable configuration of connection profiles.

## Listing Existing Connections

```bash
# Show all connection profiles
nmcli con show

# Show only active connections
nmcli con show --active
```

## Modifying an Existing Connection to Static

```bash
# Change Wired Connection 1 from DHCP to static
nmcli con mod "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses "192.168.1.100/24" \
  ipv4.gateway "192.168.1.1" \
  ipv4.dns "8.8.8.8,1.1.1.1"

# Apply the change
nmcli con up "Wired connection 1"
```

## Creating a New Static Connection

```bash
# Create a new connection named "static-eth0"
nmcli con add \
  type ethernet \
  con-name "static-eth0" \
  ifname eth0 \
  ipv4.method manual \
  ipv4.addresses "192.168.1.100/24" \
  ipv4.gateway "192.168.1.1" \
  ipv4.dns "8.8.8.8,1.1.1.1"

# Activate it
nmcli con up "static-eth0"
```

## Switching Back to DHCP

```bash
nmcli con mod "static-eth0" ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
nmcli con up "static-eth0"
```

## Adding Additional Routes

```bash
# Add a static route for 10.0.0.0/8
nmcli con mod "static-eth0" +ipv4.routes "10.0.0.0/8 192.168.1.254"
nmcli con up "static-eth0"
```

## Adding Multiple IP Addresses

```bash
# Add a second address to the connection
nmcli con mod "static-eth0" +ipv4.addresses "192.168.1.101/24"
nmcli con up "static-eth0"
```

## Verifying the Configuration

```bash
# Show all settings for a connection
nmcli con show "static-eth0"

# Verify with ip command
ip -4 addr show eth0
ip route show
```

## Setting autoconnect

```bash
# Ensure the connection activates automatically at boot
nmcli con mod "static-eth0" connection.autoconnect yes
```

## Reloading After Manual File Edits

NetworkManager stores connections in `/etc/NetworkManager/system-connections/`. If you edit these files directly:

```bash
sudo nmcli con reload
nmcli con up "static-eth0"
```

## Conclusion

`nmcli con mod` combined with `nmcli con up` is the recommended way to configure static IPs with NetworkManager. Changes are persistent immediately — no additional persistence steps needed, as NetworkManager writes the connection profile to disk automatically.
