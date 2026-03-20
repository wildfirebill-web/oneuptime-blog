# How to Configure a VLAN with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, VLAN, 802.1Q, Networking, Configuration

Description: Configure 802.1Q VLAN subinterfaces using systemd-networkd .netdev and .network files, enabling tagged VLAN traffic on a physical interface.

## Introduction

systemd-networkd manages VLANs through a `.netdev` file (to create the VLAN device) and a `.network` file (to configure the IP address). The parent interface also needs a `.network` file that references the VLAN device.

## Step 1: Create the VLAN Device (.netdev)

```ini
# /etc/systemd/network/20-vlan10.netdev

[NetDev]
Name=eth0.10
Kind=vlan

[VLAN]
Id=10
```

## Step 2: Configure the VLAN Interface (.network)

```ini
# /etc/systemd/network/20-vlan10.network
[Match]
Name=eth0.10

[Network]
Address=192.168.10.1/24
Gateway=192.168.10.254
DNS=8.8.8.8
```

## Step 3: Attach the VLAN to the Parent Interface

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
# Reference the VLAN device to attach it
VLAN=eth0.10
# Add more VLANs as needed:
# VLAN=eth0.20
# VLAN=eth0.30
```

## Apply and Verify

```bash
# Restart to create devices and apply config
systemctl restart systemd-networkd

# Verify the VLAN interface was created
ip link show eth0.10

# Check the IP address
ip addr show eth0.10

# Show full status
networkctl status eth0.10
```

## Multiple VLANs on One Interface

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
VLAN=eth0.10
VLAN=eth0.20
VLAN=eth0.30
```

Create a `.netdev` and `.network` file pair for each VLAN:

```ini
# /etc/systemd/network/21-vlan20.netdev
[NetDev]
Name=eth0.20
Kind=vlan

[VLAN]
Id=20
```

```ini
# /etc/systemd/network/21-vlan20.network
[Match]
Name=eth0.20

[Network]
Address=192.168.20.1/24
```

## DHCP on VLAN Interface

```ini
# /etc/systemd/network/20-vlan10.network
[Match]
Name=eth0.10

[Network]
DHCP=ipv4
```

## Conclusion

VLAN configuration with systemd-networkd requires three files: a `.netdev` for the VLAN device definition, a `.network` for the VLAN IP configuration, and an update to the parent interface `.network` to reference the VLAN. After applying, verify with `ip link show` and `networkctl status`.
