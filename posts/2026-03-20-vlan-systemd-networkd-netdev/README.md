# How to Configure VLANs with systemd-networkd and .netdev Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, VLAN, systemd-networkd, Netdev, 802.1Q

Description: Learn how to create and configure VLAN interfaces on Linux using systemd-networkd .netdev unit files, including IP assignment and VLAN tagging on physical interfaces.

## Introduction

`systemd-networkd` manages VLANs through two types of unit files: `.netdev` files (which create virtual network devices) and `.network` files (which configure IP addresses and routing). This approach is clean, declarative, and survives reboots without additional tooling.

## Creating a VLAN with .netdev

Create a `.netdev` file to define the VLAN interface:

```bash
sudo nano /etc/systemd/network/10-vlan100.netdev
```

```ini
[NetDev]
Name=vlan100
Kind=vlan

[VLAN]
Id=100
```

This creates a VLAN interface named `vlan100` with VLAN ID 100.

## Attaching the VLAN to a Physical Interface

Update the physical interface's `.network` file to include the VLAN:

```bash
sudo nano /etc/systemd/network/10-eth0.network
```

```ini
[Match]
Name=eth0

[Network]
DHCP=no
VLAN=vlan100
VLAN=vlan200
```

You can attach multiple VLANs by repeating the `VLAN=` directive.

## Configuring IP on the VLAN Interface

Create a `.network` file for the VLAN interface:

```bash
sudo nano /etc/systemd/network/20-vlan100.network
```

```ini
[Match]
Name=vlan100

[Network]
Address=10.100.0.1/24
Gateway=10.100.0.254
DNS=8.8.8.8
```

## Multiple VLANs Example

For VLAN 100 and VLAN 200:

```ini
# /etc/systemd/network/10-vlan100.netdev

[NetDev]
Name=vlan100
Kind=vlan
[VLAN]
Id=100

# /etc/systemd/network/10-vlan200.netdev
[NetDev]
Name=vlan200
Kind=vlan
[VLAN]
Id=200

# /etc/systemd/network/20-vlan100.network
[Match]
Name=vlan100
[Network]
Address=10.100.0.1/24

# /etc/systemd/network/20-vlan200.network
[Match]
Name=vlan200
[Network]
Address=10.200.0.1/24
```

## Applying Configuration

```bash
sudo systemctl restart systemd-networkd
```

## Verifying the VLAN Interfaces

```bash
ip link show vlan100
ip -4 addr show vlan100
networkctl status vlan100
```

## DHCP on VLAN Interface

```ini
[Network]
DHCP=ipv4
```

## Conclusion

systemd-networkd's `.netdev` and `.network` file approach makes VLAN configuration clean and reproducible. VLAN interfaces are created on boot, properly attached to their parent interfaces, and ready for IP assignment without any external scripts or network management tools.
