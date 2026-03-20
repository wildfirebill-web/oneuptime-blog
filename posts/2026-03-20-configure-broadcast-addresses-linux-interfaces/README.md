# How to Configure Broadcast Addresses in Linux Network Interfaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Linux, Broadcast, IPv4, ip command, Network Configuration

Description: Configure correct broadcast addresses on Linux network interfaces using the ip command, and understand when custom broadcast addresses are needed.

## Introduction

When you assign an IP address to a Linux interface, the kernel automatically computes the broadcast address. However, there are cases where you need to set a custom broadcast - when using non-standard addressing, VPN overlays, or legacy applications that expect a specific value.

## How the Kernel Computes the Default Broadcast

For a given network address and prefix length, the broadcast is the last address in the range (all host bits set to 1). The kernel does this automatically when you use CIDR notation.

## Assigning an IP with the Default Broadcast

```bash
# Assign 192.168.1.100/24 - kernel automatically sets broadcast to 192.168.1.255

sudo ip addr add 192.168.1.100/24 dev eth0

# Verify the broadcast address assigned
ip addr show dev eth0 | grep "inet "
# Output: inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0
```

## Assigning a Custom Broadcast Address

Use the `broadcast` keyword to override the computed value:

```bash
# Assign IP with an explicit broadcast address
sudo ip addr add 10.0.0.5/8 broadcast 10.255.255.255 dev eth0

# Assign IP with broadcast + for the entire space (non-standard)
sudo ip addr add 172.16.0.1/12 broadcast 172.31.255.255 dev eth0
```

## Using the + and - Broadcast Shortcuts

The `ip addr` command supports shorthand:

```bash
# "+" means: compute broadcast as all-ones host (standard behavior)
sudo ip addr add 192.168.5.10/24 broadcast + dev eth0

# "-" means: set broadcast to all-zeros (non-standard, rarely used)
sudo ip addr add 192.168.5.10/24 broadcast - dev eth0
```

## Checking the Current Broadcast Address

```bash
# Show all addresses and their broadcast fields
ip addr show

# Show only IPv4 with broadcast for a specific interface
ip -4 addr show dev eth0
```

## Legacy Configuration: /etc/network/interfaces

On Debian/Ubuntu systems using `ifupdown`:

```text
# /etc/network/interfaces
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    broadcast 192.168.1.255
    gateway 192.168.1.1
```

If broadcast is omitted, `ifupdown` computes it automatically.

## Netplan Configuration (Ubuntu)

Netplan does not have an explicit broadcast field - it computes the broadcast from the CIDR prefix automatically:

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
```

## Troubleshooting Wrong Broadcast

If an application is using the wrong broadcast address, check with:

```bash
# Verify the broadcast seen by applications
python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
print('Sending to 255.255.255.255')
s.sendto(b'test', ('255.255.255.255', 9999))
"
```

Then capture on the local interface to confirm the packet appears with the correct destination.

## Conclusion

Linux automatically computes correct broadcast addresses from CIDR notation. Use the `broadcast` keyword in `ip addr add` only when you need a non-standard broadcast. For most configurations, the default computation is correct and no manual override is needed.
