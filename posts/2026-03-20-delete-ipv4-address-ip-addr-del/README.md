# How to Delete an IPv4 Address from a Network Interface Using ip addr del

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, IPv4, ip command, Network Configuration

Description: Remove a specific IPv4 address from a Linux network interface using ip addr del, flush all addresses, and handle the case where the address is unknown.

## Introduction

Removing an IP address from a Linux interface is a routine task during network reconfiguration, when decommissioning a virtual IP, or when correcting a misassignment. The `ip addr del` command handles this precisely.

## Deleting a Specific Address

You must specify the exact address and prefix length that was assigned:

```bash
# Remove 192.168.1.100/24 from eth0
sudo ip addr del 192.168.1.100/24 dev eth0

# Verify it is gone
ip -4 addr show dev eth0
```

If the address was not found, you will see:

```
RTNETLINK answers: Cannot assign requested address
```

## Finding the Exact Address and Prefix

If you are unsure of the prefix:

```bash
# List all addresses with their prefix lengths
ip -4 addr show dev eth0 | grep inet
```

Use the exact `192.168.x.x/YY` string shown in the output with `ip addr del`.

## Deleting a Secondary Address

Secondary addresses (added after the primary) are deleted the same way:

```bash
sudo ip addr del 192.168.1.101/24 dev eth0
```

When you delete the primary address on an interface, all secondary addresses on the same subnet are automatically deleted too. Secondary addresses on different subnets are not affected.

## Flushing All Addresses from an Interface

```bash
# Remove all IPv4 addresses from eth0
sudo ip -4 addr flush dev eth0

# Remove ALL addresses (including IPv6) from eth0
sudo ip addr flush dev eth0
```

**Warning:** flushing all addresses will immediately break connectivity on that interface.

## Flushing by Label

If you used labels when assigning:

```bash
# Remove all addresses with label eth0:web
sudo ip addr flush dev eth0 label eth0:web
```

## Removing Persistent Configuration

If the address is configured in a persistent file, remove it there too, or it will return on reboot:

**Netplan (Ubuntu):**

```bash
sudo nano /etc/netplan/01-netcfg.yaml
# Remove the specific address from the addresses list
sudo netplan apply
```

**Debian /etc/network/interfaces:**

```bash
sudo nano /etc/network/interfaces
# Remove or comment out the relevant stanza
sudo systemctl restart networking
```

**NetworkManager:**

```bash
# Find the connection name
nmcli con show

# Remove the address
nmcli con mod "Wired connection 1" -ipv4.addresses 192.168.1.100/24
nmcli con up "Wired connection 1"
```

## Removing an Address That No Longer Exists

If an interface was already flushed or the address was never assigned, `ip addr del` will error. Check first:

```bash
# Conditional delete — only if the address exists
if ip addr show dev eth0 | grep -q "192.168.1.100"; then
    sudo ip addr del 192.168.1.100/24 dev eth0
    echo "Address removed"
else
    echo "Address not found"
fi
```

## Conclusion

`ip addr del <address>/<prefix> dev <interface>` is the precise way to remove a single address. Use `ip addr flush` to remove all at once. Always update the corresponding persistent configuration file to prevent the address from being reassigned on the next boot.
