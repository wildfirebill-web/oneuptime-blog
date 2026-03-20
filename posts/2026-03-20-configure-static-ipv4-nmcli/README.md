# How to Configure a Static IPv4 Address with nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, nmcli, NetworkManager, Static IP, IPv4, Networking

Description: Configure a static IPv4 address on a Linux network interface using nmcli (NetworkManager CLI), including gateway, DNS, and making the configuration persistent.

## Introduction

`nmcli` is the command-line interface for NetworkManager. Configuring a static IP involves modifying a connection profile's IP method, address, gateway, and DNS settings, then activating the connection.

## Set a Static IP on an Existing Connection

```bash
# Identify the connection name
nmcli connection show

# Set static IP method and address
nmcli connection modify "Wired connection 1" \
    ipv4.method manual \
    ipv4.addresses 192.168.1.100/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.dns "8.8.8.8,8.8.4.4"

# Apply changes by reactivating the connection
nmcli connection up "Wired connection 1"
```

## Create a New Static Connection from Scratch

```bash
# Create a new connection profile for eth0 with a static IP
nmcli connection add \
    type ethernet \
    con-name "static-eth0" \
    ifname eth0 \
    ipv4.method manual \
    ipv4.addresses 10.0.0.50/24 \
    ipv4.gateway 10.0.0.1 \
    ipv4.dns "1.1.1.1,8.8.8.8"

# Activate the new connection
nmcli connection up "static-eth0"
```

## Verify the Configuration

```bash
# Show assigned IP address
ip addr show eth0

# Show the connection details
nmcli connection show "static-eth0"

# Check routing
ip route show
```

## Add a Second IP Address

```bash
# Append a second IP to the same connection
nmcli connection modify "static-eth0" \
    +ipv4.addresses 10.0.0.51/24

nmcli connection up "static-eth0"
```

## Remove an IP Address

```bash
# Remove a specific IP address
nmcli connection modify "static-eth0" \
    -ipv4.addresses 10.0.0.51/24

nmcli connection up "static-eth0"
```

## Change IP Without Reconnecting (Temporary)

```bash
# Immediate IP change via ip command (not persistent)
ip addr flush dev eth0
ip addr add 10.0.0.100/24 dev eth0
ip route add default via 10.0.0.1

# For persistent changes, use nmcli modify + up
```

## Disable IPv6 While Configuring IPv4

```bash
nmcli connection modify "static-eth0" \
    ipv6.method disabled

nmcli connection up "static-eth0"
```

## Conclusion

Static IP configuration with nmcli uses `nmcli connection modify` with `ipv4.method manual`, `ipv4.addresses`, `ipv4.gateway`, and `ipv4.dns`. Activate with `nmcli connection up`. Use `nmcli connection show <name>` to verify the stored configuration.
