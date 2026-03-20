# How to Configure IPv6 in VM Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Virtual Machine, KVM, VirtualBox, VMware, Network Virtualization

Description: Configure IPv6 for virtual machines in KVM/QEMU, VirtualBox, and VMware environments, including bridged, NAT, and host-only networking modes.

## Introduction

Virtual machines present unique challenges for IPv6: the hypervisor network stack, virtual switches, and bridging all need to pass IPv6 traffic correctly. NDP (Neighbor Discovery) packets, Router Advertisements, and multicast traffic behave differently in virtualized environments. This guide covers IPv6 configuration for the most common hypervisors.

## KVM/QEMU with libvirt

```bash
# Create a network with IPv6 enabled

cat << 'EOF' > /tmp/ipv6-network.xml
<network>
  <name>ipv6-net</name>
  <forward mode='nat'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.2' end='192.168.100.254'/>
    </dhcp>
  </ip>
  <ip family='ipv6' address='fd00:cafe::/64' prefix='64'>
    <dhcp>
      <range start='fd00:cafe::2' end='fd00:cafe::ff'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-define /tmp/ipv6-network.xml
sudo virsh net-start ipv6-net
sudo virsh net-autostart ipv6-net

# Enable IPv6 on the bridge interface
sudo sysctl -w net.ipv6.conf.virbr1.disable_ipv6=0
sudo ip -6 addr add fd00:cafe::1/64 dev virbr1
```

## KVM/QEMU Bridged Networking for IPv6

```bash
# For bridged networking, the VM gets IPv6 from your physical network
# Configure bridge to pass IPv6 (should work by default)

# Create bridge
sudo ip link add br0 type bridge
sudo ip link set eth0 master br0
sudo ip link set br0 up

# Enable IPv6 on bridge
sudo sysctl -w net.ipv6.conf.br0.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.br0.accept_ra=1

# Check if bridge passes NDP/RA
sudo tcpdump -i br0 "ip6 proto 58" &
# Start VM and check if it gets an IPv6 address
```

## VirtualBox IPv6 Configuration

```bash
# VirtualBox: Enable IPv6 on host-only adapter
# GUI: File → Host Network Manager → IPv6 tab
# Or use VBoxManage:

# Create host-only network with IPv6
VBoxManage hostonlyif create  # Creates vboxnet0
VBoxManage hostonlyif ipconfig vboxnet0 \
    --ip 192.168.56.1 --netmask 255.255.255.0

# Enable IPv6 on the host-only adapter
VBoxManage hostonlyif ipconfig vboxnet0 \
    --ipv6 fd56::1 --netmasklengthv6 64

# For NAT networks with IPv6:
VBoxManage natnetwork add \
    --netname "NATNetworkv6" \
    --network "192.168.15.0/24" \
    --enable --ipv6

# Enable IPv6 on the NAT network
VBoxManage natnetwork modify \
    --netname "NATNetworkv6" \
    --ipv6 on
```

## VMware IPv6 Configuration

```bash
# VMware Workstation/Fusion: IPv6 works by default in bridged mode
# For NAT mode, configure vmnet8 for IPv6:

# Linux host: edit /etc/vmware/vmnet8/dhcpd.conf
# Add IPv6 subnet:
# subnet6 fd00:vmware::/64 {
#     range6 fd00:vmware::2 fd00:vmware::ff;
# }

# Enable IPv6 forwarding on VMware NAT interface
sudo sysctl -w net.ipv6.conf.vmnet8.disable_ipv6=0
sudo ip -6 addr add fd00:vmware::1/64 dev vmnet8

# Add radvd configuration for VMware guests
cat << 'EOF' | sudo tee /etc/radvd.conf
interface vmnet8 {
    AdvSendAdvert on;
    prefix fd00:vmware::/64 {
        AdvOnLink on;
        AdvAutonomous on;
    };
};
EOF
sudo systemctl start radvd
```

## In-VM IPv6 Troubleshooting

```bash
# Inside the VM, diagnose IPv6 issues
# Check if interface received RA from host
rdisc6 eth0

# Check if hypervisor is blocking NDP
# On the host, capture NDP traffic on the bridge/vmnet interface:
sudo tcpdump -i virbr0 "ip6 proto 58" -v

# Ensure multicast works for NDP
# Check if bridge filters multicast (common issue)
cat /sys/class/net/virbr0/bridge/multicast_snooping
# 0 = good (doesn't filter), 1 = may block NDP

# Disable multicast snooping if causing NDP issues
echo 0 | sudo tee /sys/class/net/virbr0/bridge/multicast_snooping
```

## Enabling IPv6 on VM Network Bridge (Linux)

```bash
# Ensure kernel bridge settings allow IPv6 NDP
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv6.conf.all.proxy_ndp=1

# If using ebtables, ensure IPv6/NDP is not filtered
sudo ebtables -L | grep "ipv6\|ip6"
# Should not have DROP rules for IPv6

# For LXC/LXD containers with IPv6:
lxc network set lxdbr0 ipv6.address fd00:lxd::1/64
lxc network set lxdbr0 ipv6.nat true
```

## Conclusion

IPv6 in VM environments requires IPv6 support at the hypervisor network layer, not just inside the VM. For bridged networking, IPv6 typically works automatically if the physical network provides RAs. For NAT/host-only networks, configure the virtual bridge with an IPv6 address and run `radvd` to advertise prefixes to VMs. Key troubleshooting steps: check `multicast_snooping` on bridges (disable if NDP fails), verify `accept_ra` inside VMs, and capture NDP traffic on the bridge interface to confirm RA packets are reaching the VMs.
