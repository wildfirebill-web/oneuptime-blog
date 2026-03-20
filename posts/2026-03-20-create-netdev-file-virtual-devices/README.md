# How to Create .netdev Files for Virtual Devices in systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Linux, systemd-networkd, Netdev, VLAN, Bonding

Description: Learn how to create systemd-networkd .netdev files to define virtual network devices like VLANs, bridges, bonds, and VXLAN tunnels.

---

.netdev files in systemd-networkd define virtual network devices - VLANs, bridges, bonds, VXLAN tunnels, and more - before they are configured with `.network` files.

---

## .netdev File Location and Naming

```bash
# Files are in:

/etc/systemd/network/

# Naming convention (lower number = processed first):
10-bond0.netdev
20-br0.netdev
30-vlan10.netdev
```

---

## VLAN .netdev

```ini
# /etc/systemd/network/20-vlan10.netdev
[NetDev]
Name=vlan10
Kind=vlan

[VLAN]
Id=10
```

---

## Bridge .netdev

```ini
# /etc/systemd/network/20-br0.netdev
[NetDev]
Name=br0
Kind=bridge

[Bridge]
STP=yes
ForwardDelaySec=4
```

---

## Bond .netdev

```ini
# /etc/systemd/network/20-bond0.netdev
[NetDev]
Name=bond0
Kind=bond

[Bond]
Mode=active-backup
MIIMonitorSec=1s
```

---

## VXLAN .netdev

```ini
# /etc/systemd/network/30-vxlan10.netdev
[NetDev]
Name=vxlan10
Kind=vxlan

[VXLAN]
VNI=10
Remote=192.168.1.2
Local=192.168.1.1
DestinationPort=4789
```

---

## Dummy Interface (for testing)

```ini
# /etc/systemd/network/10-dummy0.netdev
[NetDev]
Name=dummy0
Kind=dummy
```

---

## Macvlan .netdev

```ini
# /etc/systemd/network/25-macvlan0.netdev
[NetDev]
Name=macvlan0
Kind=macvlan

[MACVLAN]
Mode=bridge
```

---

## Apply and Verify

```bash
sudo systemctl restart systemd-networkd

# List all network devices
networkctl list

# Show specific device details
networkctl status vlan10
```

---

## Summary

Create `.netdev` files in `/etc/systemd/network/` to define virtual devices. The `[NetDev]` section sets the name and kind (vlan, bridge, bond, vxlan, dummy, macvlan). Add kind-specific sections like `[VLAN]`, `[Bridge]`, or `[Bond]` for device-specific parameters. Follow with a `.network` file to configure the IP and attach the device to a physical interface.
