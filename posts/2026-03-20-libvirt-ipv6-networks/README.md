# How to Configure IPv6 in libvirt Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, libvirt, KVM, Virtual Networks, DHCPv6, Linux

Description: Configure libvirt virtual networks with IPv6 subnets, DHCPv6 for guest address assignment, isolated IPv6 networks, and router advertisement configuration for VMs managed by libvirt.

## Introduction

libvirt manages virtual networks for KVM/QEMU VMs through its network configuration XML format. IPv6 is supported in libvirt networks since version 0.9.4 and includes support for DHCPv6 address assignment, static IPv6 routes, and both isolated and routed/NAT network modes. The `virsh net-*` commands manage IPv6 networks alongside IPv4.

## Create a libvirt Network with IPv6

```xml
<!-- /tmp/net-ipv6-dual.xml — Dual-stack network with DHCPv4 and DHCPv6 -->
<network>
  <name>dual-stack</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr-ds' stp='on' delay='0'/>
  <mac address='52:54:00:aa:bb:cc'/>

  <!-- IPv4 subnet with DHCP -->
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.100' end='192.168.100.200'/>
    </dhcp>
  </ip>

  <!-- IPv6 ULA subnet with DHCPv6 -->
  <ip family='ipv6' address='fd00:vm:net1::1' prefix='64'>
    <dhcp>
      <range start='fd00:vm:net1::100' end='fd00:vm:net1::200'/>
    </dhcp>
  </ip>
</network>
```

```bash
# Define, start, and autostart the network
virsh net-define /tmp/net-ipv6-dual.xml
virsh net-start dual-stack
virsh net-autostart dual-stack

# Verify
virsh net-info dual-stack
virsh net-dumpxml dual-stack
```

## IPv6-Only isolated Network

```xml
<!-- /tmp/net-ipv6-only.xml -->
<network>
  <name>ipv6-only</name>
  <!-- No forward element = isolated network -->
  <bridge name='virbr-ipv6' stp='on' delay='0'/>

  <ip family='ipv6' address='2001:db8:vms::1' prefix='64'>
    <dhcp>
      <range start='2001:db8:vms::100' end='2001:db8:vms::200'/>
      <!-- Static host reservation -->
      <host id='00:01:00:01:xx:xx:xx:xx' name='vm1' ip='2001:db8:vms::10'/>
    </dhcp>
  </ip>
</network>
```

## Routed IPv6 Network (No NAT)

```xml
<!-- /tmp/net-ipv6-routed.xml -->
<network>
  <name>ipv6-routed</name>
  <!-- Routed: host routes between VM network and physical network -->
  <forward mode='route' dev='eth0'/>
  <bridge name='virbr-rt' stp='on' delay='0'/>

  <ip family='ipv6' address='2001:db8:vms::1' prefix='64'>
    <!-- VMs get SLAAC from libvirt's radvd -->
    <!-- No dhcp element = SLAAC only -->
  </ip>

  <!-- Static route on the host will be needed: -->
  <!-- ip -6 route add 2001:db8:vms::/64 dev virbr-rt -->
</network>
```

## Manage Network with virsh

```bash
# List all networks
virsh net-list --all

# Show network details including IPv6
virsh net-dumpxml dual-stack

# Check DHCP leases (IPv4 and IPv6)
virsh net-dhcp-leases dual-stack
# Shows both DHCPv4 and DHCPv6 leases

# Add a DHCPv6 static reservation
virsh net-update dual-stack add ip-dhcp-host \
    '<host id="00:03:00:01:52:54:00:aa:bb:01" name="myvm" ip="fd00:vm:net1::50"/>' \
    --live --config

# Delete a network
virsh net-destroy dual-stack && virsh net-undefine dual-stack
```

## Router Advertisement Configuration in libvirt

```bash
# libvirt uses its built-in radvd to send Router Advertisements
# for IPv6 networks. The bridge interface acts as the IPv6 gateway.

# Check if radvd is running for the network
ps aux | grep radvd

# libvirt generates radvd.conf automatically from the network XML
# View the generated radvd config
cat /var/lib/libvirt/network/dual-stack.conf
# or
cat /run/libvirt/network/radvd-dual-stack.conf

# The RA will advertise the IPv6 prefix to VMs
# VMs generate SLAAC addresses from this prefix
```

## VM Network Attachment with IPv6

```xml
<!-- VM network interface definition in domain XML -->
<interface type='network'>
  <source network='dual-stack'/>
  <model type='virtio'/>
  <mac address='52:54:00:11:22:33'/>
</interface>
```

```bash
# Attach network to running VM
virsh attach-interface myvm network dual-stack \
    --model virtio \
    --mac 52:54:00:11:22:33 \
    --live --persistent

# Check VM's IP addresses (requires guest agent)
virsh domifaddr myvm
# Should show both IPv4 and IPv6 addresses

# Check DHCP leases for the VM
virsh net-dhcp-leases dual-stack | grep 52:54:00:11:22:33
```

## Troubleshooting libvirt IPv6 Networks

```bash
# Check bridge has IPv6 address
ip -6 addr show virbr-ds
# Should show: fd00:vm:net1::1/64

# Check radvd is sending RAs
tcpdump -i virbr-ds -n "icmp6 and icmp6[0] == 134"
# Type 134 = Router Advertisement

# Check DHCPv6 requests from VMs
tcpdump -i virbr-ds -n "udp and (port 546 or port 547)"
# 546 = DHCPv6 client, 547 = DHCPv6 server

# Restart libvirt network if radvd is not working
virsh net-destroy dual-stack
virsh net-start dual-stack

# Check libvirt logs
journalctl -u libvirtd -n 100 | grep -i ipv6
```

## Conclusion

libvirt networks support IPv6 via the `<ip family='ipv6'>` element in network XML, supporting DHCPv6 ranges, static host reservations by DUID/ID, and SLAAC via built-in radvd. Three forwarding modes work with IPv6: isolated (no forward element), routed (`mode='route'`), and NAT (`mode='nat'`). The `virsh net-dhcp-leases` command shows both IPv4 and IPv6 leases. VMs connect to IPv6 networks by being attached to a libvirt network with IPv6 configuration — the guest OS receives the IPv6 prefix via SLAAC or DHCPv6 from libvirt's built-in radvd and dnsmasq processes.
