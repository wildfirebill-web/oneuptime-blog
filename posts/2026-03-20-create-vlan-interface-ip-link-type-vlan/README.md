# How to Create a VLAN Interface with ip link type vlan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Linux, VLAN, Ip link, Network Configuration, 802.1Q

Description: Learn how to create and configure 802.1Q VLAN interfaces on Linux using the ip link command with type vlan.

---

VLAN (802.1Q) interfaces let a single physical interface carry traffic for multiple logical networks by tagging frames with a VLAN ID. Linux creates VLAN sub-interfaces using the `ip link` command.

---

## Create a VLAN Interface

```bash
# Create VLAN 10 on eth0

sudo ip link add link eth0 name eth0.10 type vlan id 10

# Bring the interface up
sudo ip link set eth0.10 up

# Assign an IP address
sudo ip addr add 192.168.10.5/24 dev eth0.10

# Add a default route via the VLAN gateway
sudo ip route add default via 192.168.10.1 dev eth0.10
```

---

## Create Multiple VLANs on One Interface

```bash
sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip link add link eth0 name eth0.30 type vlan id 30

sudo ip link set eth0.20 up
sudo ip link set eth0.30 up

sudo ip addr add 192.168.20.5/24 dev eth0.20
sudo ip addr add 192.168.30.5/24 dev eth0.30
```

---

## Verify VLAN Interfaces

```bash
ip link show type vlan
ip -d link show eth0.10  # Detailed info including VLAN ID

# Example output:
# eth0.10@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
#     vlan id 10 <REORDER_HDR>
```

---

## Make VLAN Persistent with systemd-networkd

```ini
# /etc/systemd/network/20-vlan10.netdev
[NetDev]
Name=vlan10
Kind=vlan

[VLAN]
Id=10
```

```ini
# /etc/systemd/network/21-eth0-vlan.network
[Match]
Name=eth0

[Network]
VLAN=vlan10
```

```ini
# /etc/systemd/network/30-vlan10.network
[Match]
Name=vlan10

[Network]
Address=192.168.10.5/24
Gateway=192.168.10.1
```

---

## Remove a VLAN Interface

```bash
sudo ip link set eth0.10 down
sudo ip link delete eth0.10
```

---

## Summary

Create VLAN sub-interfaces with `ip link add link <parent> name <parent>.<vid> type vlan id <vid>`. Bring them up with `ip link set up` and assign IPs with `ip addr add`. Changes made with `ip` commands are temporary - persist them with systemd-networkd `.netdev` and `.network` files.
