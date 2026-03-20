# How to Configure VLAN with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VLAN, systemd-networkd, 802.1Q, Networking, Network Configuration

Description: Configure persistent VLAN subinterfaces using systemd-networkd .netdev and .network files for declarative, reboot-safe network configuration.

## Introduction

systemd-networkd manages network interfaces through declarative configuration files stored in `/etc/systemd/network/`. VLAN configuration requires two files: a `.netdev` file to create the virtual VLAN device and a `.network` file to assign an IP address to it.

## Prerequisites

- systemd-networkd enabled (`systemctl enable --now systemd-networkd`)
- Root access

## Step 1: Create the .netdev File for the VLAN

The `.netdev` file defines the VLAN virtual device:

```ini
# /etc/systemd/network/10-vlan100.netdev
[NetDev]
# Name of the VLAN interface
Name=eth0.100
Kind=vlan

[VLAN]
# VLAN ID
Id=100
```

## Step 2: Create the .network File for the VLAN

The `.network` file assigns an IP address and configures routing:

```ini
# /etc/systemd/network/10-vlan100.network
[Match]
Name=eth0.100

[Network]
Address=192.168.100.1/24
Gateway=192.168.100.254
DNS=8.8.8.8
DNS=8.8.4.4
```

## Step 3: Configure the Parent Interface

The parent interface also needs a `.network` file to declare the VLAN it carries:

```ini
# /etc/systemd/network/05-eth0.network
[Match]
Name=eth0

[Network]
# Tell networkd that eth0 carries a VLAN
VLAN=eth0.100
# No IP on the parent (trunk mode)
```

## Apply the Configuration

```bash
# Restart networkd to apply changes
systemctl restart systemd-networkd

# Verify the VLAN interface
ip addr show eth0.100

# Check networkd status
networkctl status eth0.100
```

## Multiple VLANs Configuration

```ini
# /etc/systemd/network/10-vlan10.netdev
[NetDev]
Name=eth0.10
Kind=vlan

[VLAN]
Id=10
```

```ini
# /etc/systemd/network/10-vlan10.network
[Match]
Name=eth0.10

[Network]
Address=10.10.0.1/24
```

```ini
# /etc/systemd/network/10-vlan20.netdev
[NetDev]
Name=eth0.20
Kind=vlan

[VLAN]
Id=20
```

```ini
# /etc/systemd/network/10-vlan20.network
[Match]
Name=eth0.20

[Network]
Address=10.20.0.1/24
```

```ini
# /etc/systemd/network/05-eth0.network (updated with multiple VLANs)
[Match]
Name=eth0

[Network]
VLAN=eth0.10
VLAN=eth0.20
```

## Verify with networkctl

```bash
# List all managed interfaces
networkctl list

# Show detailed status of the VLAN interface
networkctl status eth0.100

# Check networkd logs
journalctl -u systemd-networkd -f
```

## DHCP VLAN with systemd-networkd

```ini
# /etc/systemd/network/10-vlan200.network
[Match]
Name=eth0.200

[Network]
DHCP=ipv4
```

## Conclusion

systemd-networkd manages VLANs through pairs of `.netdev` (device definition) and `.network` (IP configuration) files. The parent interface's `.network` file must list the VLANs it carries with `VLAN=<name>` lines. Changes are persistent by default and take effect after `systemctl restart systemd-networkd`.
