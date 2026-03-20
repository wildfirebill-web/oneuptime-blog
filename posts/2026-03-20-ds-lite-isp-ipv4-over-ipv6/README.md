# How to Configure DS-Lite for ISP IPv4 Over IPv6 Tunneling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DS-Lite, IPv6, IPv4, ISP, Tunneling, AFTR, B4

Description: Configure DS-Lite (Dual-Stack Lite) for ISP deployments, setting up the AFTR (carrier NAT44 over IPv6 tunnel) and B4 (customer-premises softwire concentrator) for IPv4 connectivity over IPv6-only...

## Introduction

DS-Lite (RFC 6333) allows ISPs to provide IPv4 connectivity over IPv6-only access networks. Customer-premises equipment (CPE) runs the B4 (Basic Bridging BroadBand) element which encapsulates IPv4 traffic in IPv6 tunnels. The ISP runs AFTR (Address Family Transition Router) which decapsulates and NAT44s the traffic to the IPv4 internet.

## Architecture

```text
IPv4 Client → B4 (CPE) →[IPv4-in-IPv6 tunnel]→ AFTR (ISP) → IPv4 Internet
              [encap]        [IPv6-only access]    [NAT44]
```

- **B4**: Located at customer premises; encapsulates private IPv4 in IPv6 (softwire)
- **AFTR**: Located at ISP; decapsulates IPv6, performs NAT44 to public IPv4

## Setting Up AFTR on Linux

```bash
# Install softwire/MAP-E support

# Option 1: Use LW4over6 or AFTR implementation
# Option 2: Use kernel ip6tnl with iptables

# Create the AFTR tunnel endpoint
# Each B4 gets a dedicated IPv6 address for its softwire

# Example AFTR configuration for one B4:
# B4 IPv6 address: 2001:db8:1:1::100
# AFTR IPv6 address: 2001:db8:ffff::1
# B4's private IPv4: 192.168.1.0/24

# Create IPv6-in-IPv4 tunnel (ip6tnl) for AFTR
sudo ip tunnel add aftr0 mode ip4ip6 \
  remote 2001:db8:1:1::100 \
  local 2001:db8:ffff::1 \
  dev eth0

sudo ip link set aftr0 up
sudo ip addr add 192.0.0.1/31 dev aftr0

# Route the B4's private subnet through the tunnel
sudo ip route add 192.168.1.0/24 dev aftr0

# NAT44: translate B4's private IPv4 to public IPv4
sudo iptables -t nat -A POSTROUTING -i aftr0 -o eth1 -j MASQUERADE

# Enable IPv6 and IPv4 forwarding
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.ip_forward=1
```

## Setting Up B4 on CPE (Linux)

```bash
# B4 runs on the customer CPE (e.g., OpenWrt, Linux router)
# The CPE gets a native IPv6 address from the ISP

# CPE's IPv6 address: 2001:db8:1:1::100
# AFTR's IPv6 address: 2001:db8:ffff::1

# Create IPv4-in-IPv6 softwire tunnel to AFTR
sudo ip tunnel add b4-aftr mode ip4ip6 \
  remote 2001:db8:ffff::1 \
  local 2001:db8:1:1::100 \
  dev eth0   # WAN interface with IPv6

sudo ip link set b4-aftr up
sudo ip addr add 192.0.0.2/31 dev b4-aftr

# Route all IPv4 traffic through the B4 tunnel to AFTR
sudo ip route add default dev b4-aftr

# LAN devices use the CPE as their IPv4 gateway
# No NAT needed on B4 if AFTR does the NAT44
```

## AFTR IPv6 Address Discovery

The B4 discovers the AFTR address via DHCPv6:

```text
# DHCPv6 option 64: AFTR-Name
# /etc/dhcp/dhcpd6.conf (on ISP DHCPv6 server)

subnet6 2001:db8:1::/48 {
    range6 2001:db8:1:1::100 2001:db8:1:1::ffff;
    option dhcp6.aftr-name "aftr.isp.example.com";
}
```

The B4 performs DNS lookup for the AFTR name and establishes the softwire.

## Verifying DS-Lite

```bash
# On B4 (CPE):
# Check tunnel is up
ip tunnel show b4-aftr

# Check default route via tunnel
ip route show default
# Expected: default dev b4-aftr

# Test IPv4 connectivity through AFTR
ping 8.8.8.8

# Test IPv6 native connectivity
ping6 2001:4860:4860::8888

# On AFTR (ISP side):
# Check active softwires
ip tunnel show

# Monitor NAT44 translations
conntrack -L | head -20

# Check packet forwarding stats
ip -s link show aftr0
```

## Port Sharing Considerations

Since multiple B4 customers share AFTR public IPv4 addresses:

```bash
# AFTR can use port ranges to identify customers
# Using iptables SNAT with port ranges:

# Customer 1: ports 1024-2047
iptables -t nat -A POSTROUTING -s 2001:db8:1:1::100 \
  -j SNAT --to-source 203.0.113.1:1024-2047

# Customer 2: ports 2048-3071
iptables -t nat -A POSTROUTING -s 2001:db8:1:1::101 \
  -j SNAT --to-source 203.0.113.1:2048-3071
```

## Firewall on AFTR

```bash
# Allow softwire traffic (IPv4-in-IPv6 protocol 4)
sudo ip6tables -A INPUT -p ipv6-nonxt -j ACCEPT   # if using ip6tnl
sudo ip6tables -A FORWARD -i b4-* -j ACCEPT

# Rate limit per B4 to prevent abuse
sudo iptables -A FORWARD -i aftr0 -m limit --limit 1000/sec -j ACCEPT
sudo iptables -A FORWARD -i aftr0 -j DROP
```

## Conclusion

DS-Lite provides IPv4 connectivity over IPv6-only ISP access networks. The B4 on the CPE encapsulates IPv4 traffic in IPv6 softwires directed at the AFTR. The AFTR decapsulates and performs NAT44 using shared public IPv4 addresses. B4s discover the AFTR address via DHCPv6 option 64 (AFTR-Name). On Linux, `ip tunnel add mode ip4ip6` creates the softwire on both B4 and AFTR. Port-range SNAT on the AFTR allows port-based subscriber identification when sharing public IPs across multiple customers.
