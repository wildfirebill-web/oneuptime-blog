# How to Configure IPv6 NAT Networking in VirtualBox

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, VirtualBox, NAT, NPTv6, Virtual Networking, Desktop Virtualization

Description: Configure IPv6 NAT and NAT Network modes in VirtualBox to provide IPv6 connectivity to virtual machines when using ULA prefixes, including NPTv6 translation and port forwarding.

## Introduction

VirtualBox's NAT and NAT Network modes provide IPv6 connectivity to VMs when bridged networking is not available or desired. NAT mode uses NPTv6 (Network Prefix Translation) to translate between the VM's private IPv6 prefix and the host's public IPv6 address. NAT Network mode provides DHCPv6 and allows multiple VMs to communicate with each other and the outside world.

## VirtualBox NAT Mode with IPv6

```bash
# NAT mode (default VirtualBox network) - single VM, IPv6 via host

# Check if IPv6 is enabled for NAT on a VM

VBoxManage showvminfo "MyVM" | grep -E "NIC 1|IPv6"

# Enable IPv6 for NAT adapter
VBoxManage modifyvm "MyVM" \
    --nic1 nat

# VirtualBox NAT automatically provides IPv6 if the host has IPv6
# The VM gets:
# - Link-local: fe80::1/64 (always)
# - If host has IPv6: the VM can reach IPv6 destinations

# Port forwarding over IPv6 (NAT mode)
VBoxManage modifyvm "MyVM" \
    --natpf1 "ssh-ipv6,tcp,[::1],2222,[],22"
# Connect: ssh -p 2222 -6 ::1
```

## NAT Network with IPv6

```bash
# Create NAT Network with IPv6 DHCP
VBoxManage natnetwork add \
    --netname NatNet1 \
    --network "10.0.2.0/24" \
    --ipv6 on \
    --ipv6prefix "fd17:625c:f037:cafe::/64" \
    --enable

# Verify the NAT Network
VBoxManage natnetwork list
VBoxManage natnetwork modify NatNet1 --port-forward-6 "ssh6:tcp:[]:2222:[fd17:625c:f037:cafe::100]:22"

# Start the NAT Network
VBoxManage natnetwork start NatNet1

# Attach VM to NAT Network
VBoxManage modifyvm "MyVM" \
    --nic1 natnetwork \
    --nat-network1 "NatNet1"

VBoxManage startvm "MyVM" --type headless
```

## Inside the VM: Verify IPv6 from NAT Network

```bash
# Inside the VM (connected to NAT Network)

# Check IPv6 address from DHCPv6
ip -6 addr show
# Expected:
# inet6 fd17:625c:f037:cafe::100/64 scope global dynamic

# Check default route
ip -6 route show
# Expected:
# default via fd17:625c:f037:cafe::1 dev eth0

# Test IPv6 connectivity
ping6 fd17:625c:f037:cafe::1    # Gateway
ping6 2001:4860:4860::8888      # External IPv6 (if host has IPv6)
```

## NPTv6 for IPv6-to-IPv6 Translation

```bash
# On the host: configure NPTv6 to translate VirtualBox NAT Network
# ULA prefix (fd00::/8) to public IPv6 prefix

# Enable IPv6 forwarding on host
sysctl -w net.ipv6.conf.all.forwarding=1

# Install nft or use ip6tables for NPTv6
# Using nftables (recommended)
cat > /etc/nftables.d/ipv6-nat.nft << 'EOF'
table ip6 nat {
    chain PREROUTING {
        type nat hook prerouting priority -100;
        # Translate incoming public IPv6 to ULA for VMs
        ip6 daddr 2001:db8::100/128 dnat to fd17:625c:f037:cafe::100
    }
    chain POSTROUTING {
        type nat hook postrouting priority 100;
        # NPTv6: translate VMs' ULA to public IPv6
        ip6 saddr fd17:625c:f037:cafe::/64 oifname "eth0" snat to 2001:db8::100
    }
}
EOF
nft -f /etc/nftables.d/ipv6-nat.nft
```

## Host-Only Network with IPv6 (No NAT)

```bash
# For VM-to-host and VM-to-VM IPv6 without NAT:
# Use Host-Only adapter with IPv6

# Create host-only interface
VBoxManage hostonlyif create
# Returns: Interface 'vboxnet0' was successfully created

# Configure IPv6 on host-only interface
VBoxManage hostonlyif ipconfig vboxnet0 \
    --ipv6 "fd00:vbox::1" \
    --netmasklengthv6 64

# Verify
VBoxManage list hostonlyifs | grep -A10 "vboxnet0"

# Attach VM to host-only
VBoxManage modifyvm "MyVM" \
    --nic2 hostonly \
    --hostonlyadapter2 vboxnet0

# Inside VM: configure static IPv6 on second adapter
# ip -6 addr add fd00:vbox::10/64 dev eth1
# ip -6 route add fd00:vbox::/64 dev eth1
```

## VirtualBox VM with Multiple IPv6 Networks

```bash
# VM with NAT (IPv4 internet + IPv6 if host has it)
# plus Host-Only (IPv6 management)

VBoxManage modifyvm "MyVM" \
    --nic1 nat                          # Internet (NAT)
    --nic2 hostonly \
    --hostonlyadapter2 vboxnet0         # Management (host-only IPv6)

# NIC1: DHCP IPv4 + IPv6 from host
# NIC2: Static IPv6 fd00:vbox::10/64 - direct host access
```

## Testing IPv6 NAT Connectivity

```bash
# From host: test NAT Network gateway
ping6 fd17:625c:f037:cafe::1

# From VM: test external IPv6 through NAT
# (only works if host has IPv6 internet)
curl -6 https://ipv6.google.com/

# Port forward test
ssh -p 2222 -6 "::1"   # Connect to VM's SSH via IPv6 loop

# Check VirtualBox DHCP leases for NAT Network
VBoxManage dhcpserver findlease \
    --network=NatNet1 \
    --mac-address=08:00:27:aa:bb:cc
```

## Conclusion

VirtualBox provides IPv6 through NAT mode (which relays IPv6 from the host) and NAT Network mode (which includes DHCPv6 and a configurable ULA prefix). The NAT Network mode with `--ipv6 on` and `--ipv6prefix` creates a complete IPv6 subnet with DHCPv6 for VMs. Host-Only networking with IPv6 is the simplest option for host-VM communication using ULA addresses without NAT. For full public IPv6 access, bridged networking is preferred over NAT, as it connects VMs directly to the physical IPv6 network. NPTv6 can be configured on the host using nftables to translate between NAT Network ULA addresses and public IPv6 addresses.
