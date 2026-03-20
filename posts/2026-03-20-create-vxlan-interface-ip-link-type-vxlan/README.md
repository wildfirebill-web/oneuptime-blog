# How to Create a VXLAN Interface with ip link type vxlan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Linux, VXLAN, ip link, Overlay Network, Tunneling

Description: Learn how to create and configure VXLAN overlay network interfaces on Linux using the ip link command with type vxlan.

---

VXLAN (Virtual Extensible LAN) encapsulates Layer 2 Ethernet frames within UDP packets, creating overlay networks that span physical network boundaries. Linux supports VXLAN natively via the `ip` command.

---

## Create a VXLAN Interface

```bash
# Create VXLAN tunnel with VNI 10
sudo ip link add vxlan10 type vxlan   id 10   remote 192.168.1.2   local 192.168.1.1   dev eth0   dstport 4789

# Bring the interface up
sudo ip link set vxlan10 up

# Assign an IP address
sudo ip addr add 10.10.10.1/24 dev vxlan10
```

---

## Multicast VXLAN (no specific remote)

```bash
sudo ip link add vxlan20 type vxlan   id 20   group 239.1.1.1   dev eth0   dstport 4789

sudo ip link set vxlan20 up
sudo ip addr add 10.20.0.1/24 dev vxlan20
```

---

## View VXLAN Details

```bash
# Show VXLAN interfaces
ip -d link show type vxlan

# Example output:
# vxlan10: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     vxlan id 10 remote 192.168.1.2 local 192.168.1.1
#     srcport 0 0 dstport 4789
```

---

## Add FDB (Forwarding Database) Entries

```bash
# Add remote VTEP address for a specific MAC
bridge fdb add 00:00:00:00:00:00 dev vxlan10 dst 192.168.1.3

# View FDB entries
bridge fdb show dev vxlan10
```

---

## Persist with systemd-networkd

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

```ini
# /etc/systemd/network/31-vxlan10.network
[Match]
Name=vxlan10

[Network]
Address=10.10.10.1/24
```

---

## Remove a VXLAN Interface

```bash
sudo ip link set vxlan10 down
sudo ip link delete vxlan10
```

---

## Summary

Create VXLAN interfaces with `ip link add type vxlan id <VNI>`, specifying `remote` and `local` for unicast or `group` for multicast. The standard destination port is 4789. Use `ip -d link show type vxlan` to inspect configuration and `bridge fdb` to manage forwarding entries. Persist configurations with systemd-networkd `.netdev` files.
