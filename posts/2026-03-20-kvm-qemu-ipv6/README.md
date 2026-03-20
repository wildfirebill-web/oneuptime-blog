# How to Configure IPv6 in KVM/QEMU

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, KVM, QEMU, Linux, Virtualization, Virtual Networking

Description: Configure IPv6 networking in KVM/QEMU virtual machines using bridge, NAT, and macvtap network modes, with cloud-init and manual guest OS IPv6 configuration.

## Introduction

KVM/QEMU virtual machines access IPv6 networks through virtual networking modes: bridged (VM on same L2 as host), NAT (via virbr0 MASQUERADE with IPv6 prefix), and macvtap/SR-IOV for direct IPv6. Bridged networking is the simplest for IPv6 - VMs receive SLAAC addresses from the physical router just like physical machines.

## Bridged Networking for IPv6

```bash
# Create a bridge on the host for VMs to share the physical IPv6 network

# Using ip commands

ip link add name br0 type bridge
ip link set br0 up
ip link set eth0 master br0

# Move host IPv6 address from eth0 to bridge
ip -6 addr flush dev eth0
ip -6 addr add 2001:db8::10/64 dev br0
ip -6 route add default via 2001:db8::1 dev br0

# Or using systemd-networkd (persistent)
# /etc/systemd/network/br0.netdev
# /etc/systemd/network/br0.network
```

```xml
<!-- VM definition with bridge networking (virsh) -->
<!-- /tmp/vm-net.xml -->
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
</interface>
```

```bash
# Attach bridge network to VM
virsh attach-interface myvm bridge br0 --model virtio --current --live
# VM gets SLAAC IPv6 from physical network router
```

## QEMU Command Line with IPv6 Networking

```bash
# Start a VM with bridged networking over IPv6
qemu-system-x86_64 \
    -name myvm \
    -m 2048 \
    -drive file=/var/lib/libvirt/images/myvm.qcow2 \
    -netdev bridge,id=net0,br=br0 \
    -device virtio-net-pci,netdev=net0,mac=52:54:00:ab:cd:ef \
    -nographic

# TAP-based networking for more control
ip tuntap add dev tap0 mode tap
ip link set tap0 master br0
ip link set tap0 up

qemu-system-x86_64 \
    -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-pci,netdev=net0 \
    ...
```

## libvirt Network with IPv6 (NAT Mode)

```xml
<!-- /tmp/ipv6nat-network.xml -->
<network>
  <name>ipv6nat</name>
  <forward mode='nat'>
    <nat>
      <!-- IPv6 NAT (NPTv6 / masquerade) -->
      <address start='2001:db8:nat::1' end='2001:db8:nat::ffff'/>
    </nat>
  </forward>
  <bridge name='virbr1' stp='on' delay='0'/>
  <ip family='ipv4' address='192.168.200.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.200.100' end='192.168.200.200'/>
    </dhcp>
  </ip>
  <ip family='ipv6' address='fd00:db8::1' prefix='64'>
    <dhcp>
      <range start='fd00:db8::100' end='fd00:db8::200'/>
    </dhcp>
  </ip>
</network>
```

```bash
# Define and start the network
virsh net-define /tmp/ipv6nat-network.xml
virsh net-start ipv6nat
virsh net-autostart ipv6nat

# Attach VM to this network
virsh attach-interface myvm network ipv6nat --model virtio --current
```

## Configure IPv6 in a KVM Guest (Linux)

```bash
# Inside the VM (SSH over IPv4 initially, then switch to IPv6)

# Check if SLAAC address was assigned
ip -6 addr show
# Should show a 2001:db8::/64 or fd00::/64 address from SLAAC/DHCPv6

# Static IPv6 in the guest (Ubuntu/Netplan)
cat > /etc/netplan/01-ipv6.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens3:
      addresses:
        - 2001:db8::100/64
      gateway6: 2001:db8::1
      nameservers:
        addresses:
          - 2001:db8::53
EOF
netplan apply

# Verify
ping6 2001:db8::1
```

## Cloud-init with IPv6 for KVM VMs

```yaml
# user-data for cloud-init (NoCloud datasource)

#cloud-config
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: true
      dhcp6: true    # Enable DHCPv6
      # Or static:
      addresses:
        - 2001:db8::100/64
      gateway6: 2001:db8::1
```

```bash
# Create cloud-init seed disk
cloud-localds seed.img user-data meta-data

# Start VM with cloud-init
qemu-system-x86_64 \
    -drive file=ubuntu-22.04.qcow2,format=qcow2 \
    -drive file=seed.img,format=raw \
    -netdev bridge,id=net0,br=br0 \
    -device virtio-net-pci,netdev=net0 \
    -m 2048 -nographic
```

## IPv6 Firewall for KVM Host

```bash
# Allow IPv6 forwarding between VMs and physical network
echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.conf
sysctl -p

# For bridged VMs: IPv6 flows through the bridge without NAT
# For NAT VMs: NPTv6 masquerade
ip6tables -t nat -A POSTROUTING -s fd00:db8::/64 -o eth0 -j MASQUERADE

# Allow established traffic back to VMs
ip6tables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
ip6tables -A FORWARD -s fd00:db8::/64 -j ACCEPT
```

## Conclusion

KVM/QEMU VMs access IPv6 most simply through bridged networking, where VMs appear directly on the physical IPv6 network and receive SLAAC addresses from the connected router. The libvirt XML format supports IPv6 subnets in the `<ip family='ipv6'>` element for NAT and isolated networks with DHCPv6. Inside the guest, IPv6 is configured identically to bare-metal systems using netplan, NetworkManager, or systemd-networkd. Enable `net.ipv6.conf.all.forwarding=1` on the KVM host for routing between VM networks and the physical network.
