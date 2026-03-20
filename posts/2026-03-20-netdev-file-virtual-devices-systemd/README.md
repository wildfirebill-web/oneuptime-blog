# How to Create a .netdev File for Virtual Devices in systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, .netdev file, VLAN, Bridge, Bond, VXLAN, Networking

Description: Create .netdev files for systemd-networkd to define virtual network devices such as VLANs, bridges, bonds, VXLANs, and GRE tunnels declaratively.

## Introduction

`.netdev` files define virtual network devices that systemd-networkd creates at startup. They live in `/etc/systemd/network/` alongside `.network` files. A `.netdev` file specifies the device name, kind (vlan, bridge, bond, vxlan, etc.), and device-specific parameters.

## .netdev File for a VLAN

```ini
# /etc/systemd/network/20-vlan10.netdev
[NetDev]
Name=eth0.10
Kind=vlan

[VLAN]
Id=10
```

```ini
# /etc/systemd/network/20-vlan10.network
[Match]
Name=eth0.10

[Network]
Address=192.168.10.1/24
```

## .netdev File for a Bridge

```ini
# /etc/systemd/network/10-br0.netdev
[NetDev]
Name=br0
Kind=bridge

[Bridge]
STP=yes
ForwardDelaySec=4
```

```ini
# /etc/systemd/network/10-br0.network
[Match]
Name=br0

[Network]
Address=192.168.1.1/24
```

## .netdev File for a Bond

```ini
# /etc/systemd/network/10-bond0.netdev
[NetDev]
Name=bond0
Kind=bond

[Bond]
Mode=active-backup
MIIMonitorSec=100ms
```

```ini
# /etc/systemd/network/10-bond0.network
[Match]
Name=bond0

[Network]
DHCP=ipv4
```

## .netdev File for a VXLAN

```ini
# /etc/systemd/network/30-vxlan100.netdev
[NetDev]
Name=vxlan100
Kind=vxlan

[VXLAN]
VNI=100
Remote=10.0.0.2
Local=10.0.0.1
DestinationPort=4789
```

## .netdev File for a GRE Tunnel

```ini
# /etc/systemd/network/40-gre1.netdev
[NetDev]
Name=gre1
Kind=gre

[Tunnel]
Remote=203.0.113.2
Local=203.0.113.1
TTL=64
```

## Apply Virtual Device Configuration

```bash
# Restart systemd-networkd to create .netdev devices
systemctl restart systemd-networkd

# Verify devices were created
ip link show
networkctl list
```

## File Processing Order

```
/etc/systemd/network/
  10-br0.netdev    ← bridge (created first)
  10-br0.network   ← bridge network config
  20-eth0.network  ← physical interface attached to bridge
  20-vlan10.netdev ← VLAN (depends on eth0)
  20-vlan10.network
```

`.netdev` files are processed before `.network` files — devices must exist before they can be configured.

## Conclusion

`.netdev` files define virtual network devices in systemd-networkd. Use `[NetDev]` with `Name` and `Kind`, then add a device-specific section (`[VLAN]`, `[Bridge]`, `[Bond]`, `[VXLAN]`, `[Tunnel]`). Pair each `.netdev` with a corresponding `.network` file for addressing. Apply with `systemctl restart systemd-networkd`.
