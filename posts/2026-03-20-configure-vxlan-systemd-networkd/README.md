# How to Configure a VXLAN with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, VXLAN, Overlay Network, Networking, Configuration

Description: Configure a VXLAN interface using systemd-networkd .netdev and .network files for persistent overlay network setup across reboots.

## Introduction

systemd-networkd creates VXLAN interfaces via `.netdev` files with `Kind=vxlan`. The VXLAN Network Identifier (VNI), remote endpoint, and local address are configured in the `[VXLAN]` section. A corresponding `.network` file assigns the overlay IP address.

## Create the VXLAN Device (.netdev)

```ini
# /etc/systemd/network/50-vxlan100.netdev
[NetDev]
Name=vxlan100
Kind=vxlan

[VXLAN]
VNI=100
Remote=10.0.0.2
Local=10.0.0.1
DestinationPort=4789
```

## Configure the VXLAN Overlay IP (.network)

```ini
# /etc/systemd/network/50-vxlan100.network
[Match]
Name=vxlan100

[Network]
Address=192.168.100.1/24
```

## Apply and Verify

```bash
# Restart to create VXLAN
systemctl restart systemd-networkd

# Verify the VXLAN interface
ip -d link show vxlan100

# Check overlay IP
ip addr show vxlan100

# Test connectivity to remote overlay IP
ping -c 3 192.168.100.2
```

## Remote Host Configuration (Host B)

```ini
# /etc/systemd/network/50-vxlan100.netdev
[NetDev]
Name=vxlan100
Kind=vxlan

[VXLAN]
VNI=100
Remote=10.0.0.1
Local=10.0.0.2
DestinationPort=4789
```

```ini
# /etc/systemd/network/50-vxlan100.network
[Match]
Name=vxlan100

[Network]
Address=192.168.100.2/24
```

## VXLAN with Multicast Group

```ini
# /etc/systemd/network/50-vxlan100.netdev
[NetDev]
Name=vxlan100
Kind=vxlan

[VXLAN]
VNI=100
Group=239.1.1.1
DestinationPort=4789
```

## VXLAN Attached to a Bridge

After creating the VXLAN device, attach it to a bridge:

```ini
# /etc/systemd/network/50-vxlan100.network
[Match]
Name=vxlan100

[Network]
# No IP — VXLAN is a bridge member
Bridge=br0
```

## VXLAN .netdev Options Reference

| Key | Description |
|---|---|
| `VNI` | VXLAN Network Identifier (1-16777215) |
| `Remote` | Remote VTEP IP (unicast) |
| `Local` | Local VTEP IP |
| `Group` | Multicast group IP |
| `DestinationPort` | UDP port (4789 = IANA standard) |
| `TOS` | IP TOS for outer header |
| `TTL` | IP TTL for outer header |

## Conclusion

VXLAN with systemd-networkd uses a `.netdev` file specifying `Kind=vxlan`, the VNI, and endpoint IPs, plus a `.network` file for the overlay IP. This provides persistent VXLAN configuration managed entirely by systemd without manual scripts or `ip link` commands at boot.
