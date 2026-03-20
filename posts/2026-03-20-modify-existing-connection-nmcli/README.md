# How to Modify an Existing Connection with nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, nmcli, NetworkManager, Configuration, Networking

Description: Modify existing NetworkManager connection profiles using nmcli to change IP addresses, DNS, gateways, interface bindings, and other settings without recreating the profile.

## Introduction

`nmcli connection modify` changes settings in an existing NetworkManager connection profile. Changes are stored immediately but take effect only after reactivating the connection with `nmcli connection up`.

## Basic Modify Syntax

```bash
# General syntax
nmcli connection modify <con-name> <property> <value>

# List available connections to find the correct name
nmcli connection show
```

## Change IP Address

```bash
# Change IP (replaces all existing addresses)
nmcli connection modify "myconn" \
    ipv4.addresses "10.0.0.20/24"

nmcli connection up "myconn"
```

## Change Gateway

```bash
nmcli connection modify "myconn" \
    ipv4.gateway "10.0.0.1"

nmcli connection up "myconn"
```

## Change DNS Servers

```bash
nmcli connection modify "myconn" \
    ipv4.dns "8.8.8.8 1.1.1.1"

nmcli connection up "myconn"
```

## Switch Between DHCP and Static

```bash
# Switch to static IP
nmcli connection modify "myconn" \
    ipv4.method manual \
    ipv4.addresses "192.168.1.50/24" \
    ipv4.gateway "192.168.1.1"

# Switch back to DHCP
nmcli connection modify "myconn" \
    ipv4.method auto \
    ipv4.addresses "" \
    ipv4.gateway ""

nmcli connection up "myconn"
```

## Change the Interface Binding

```bash
# Rebind connection to a different interface
nmcli connection modify "myconn" \
    connection.interface-name eth1

nmcli connection up "myconn"
```

## Enable or Disable Auto-Connect

```bash
# Disable auto-connect (manual activation only)
nmcli connection modify "myconn" \
    connection.autoconnect no

# Re-enable auto-connect
nmcli connection modify "myconn" \
    connection.autoconnect yes
```

## Change the MTU

```bash
nmcli connection modify "myconn" \
    ethernet.mtu 9000

nmcli connection up "myconn"
```

## Rename a Connection

```bash
nmcli connection modify "myconn" \
    connection.id "new-name"
```

## Verify Changes Before Applying

```bash
# Show all settings after modification (before activating)
nmcli connection show "myconn" | grep ipv4

# Then activate
nmcli connection up "myconn"
```

## Conclusion

`nmcli connection modify <name> <property> <value>` updates stored connection settings. The `+` prefix appends values, `-` removes specific values, and plain assignment replaces them. Always follow `modify` with `nmcli connection up <name>` to apply changes to the running system.
