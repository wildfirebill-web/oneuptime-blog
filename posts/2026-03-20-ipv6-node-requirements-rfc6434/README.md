# How to Understand IPv6 Node Requirements (RFC 6434)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, RFC 6434, Compliance, Standard, Node Requirements, Networking

Description: Understand the IPv6 node requirements defined in RFC 6434, which specifies mandatory and recommended features that IPv6-capable devices must implement.

---

RFC 6434 ("IPv6 Node Requirements") defines the minimum set of IPv6 features that all IPv6 nodes (hosts and routers) must implement to be considered IPv6-compliant. Understanding these requirements is essential for compliance testing and deployment planning.

## RFC 6434 Key Requirements Overview

```text
RFC 6434 (obsoletes RFC 4294) defines requirements for:
- IPv6 addressing
- ICMPv6 (RFC 4443)
- Neighbor Discovery (RFC 4861)
- Stateless Address Autoconfiguration - SLAAC (RFC 4862)
- Path MTU Discovery (RFC 8201)
- IPv6 fragmentation and reassembly
- DNS support (RFC 3596 - AAAA records)
- Privacy Extensions (RFC 4941) - recommended
- Multicast Listener Discovery (RFC 3810)
```

## Core IPv6 Addressing Requirements (MUST implement)

```bash
# RFC 6434 §4 - Addressing:

# - MUST support link-local addresses (FE80::/10)
# - MUST support loopback address (::1)
# - MUST support multicast addresses
# - MUST support unicast addresses

# Verify addressing on Linux (RFC 6434 compliant)
ip -6 addr show

# Link-local should always be present
ip -6 addr show | grep "fe80"

# Loopback should be present
ip -6 addr show lo | grep "::1"
```

## ICMPv6 Requirements (MUST implement)

```bash
# RFC 6434 requires ICMPv6 (RFC 4443)
# Nodes MUST implement and respond to:
# - Echo Request (type 128) and Reply (type 129)
# - Neighbor Solicitation (type 135) / Advertisement (type 136)
# - Router Solicitation (type 133)
# - Packet Too Big (type 2) for Path MTU Discovery

# Test ICMPv6 echo
ping6 ::1
ping6 fe80::1%eth0

# Test ICMPv6 Packet Too Big handling (PMTU)
ping6 -s 1500 2001:db8::host  # Should receive Packet Too Big if MTU < 1500

# Do NOT block ICMPv6 (unlike IPv4 ICMP, ICMPv6 is mandatory for IPv6)
# Firewall rule to allow all necessary ICMPv6
sudo ip6tables -A INPUT -p icmpv6 -j ACCEPT
```

## Neighbor Discovery Requirements

```bash
# RFC 6434 §4.1 - Neighbor Discovery (RFC 4861) MUST implement:
# - Neighbor Solicitation/Advertisement
# - Router Solicitation/Advertisement
# - Duplicate Address Detection (DAD)
# - Address resolution (replaces ARP)

# Check Neighbor Discovery is working
ip -6 neigh show

# Verify DAD is running on new addresses
# (tentative state appears briefly)
ip -6 addr show | grep "tentative"

# Test NDP
sudo arping6 -I eth0 2001:db8::gateway
```

## Path MTU Discovery Requirements

```bash
# RFC 6434 requires Path MTU Discovery (RFC 8201)
# - Nodes MUST handle "Packet Too Big" ICMPv6 messages
# - MUST NOT send packets larger than PMTU

# Check current PMTU
ip -6 route show cache | grep "mtu"

# Test PMTU discovery
tracepath6 2001:db8::destination

# Ensure PMTU ICMPv6 is not filtered
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
sudo ip6tables -A OUTPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
```

## DNS Requirements (RFC 6434)

```bash
# RFC 6434 requires IPv6 DNS support:
# - MUST support AAAA records
# - Recommended: Use DNS for address lookup (not hardcoded IPs)

# Test AAAA record resolution
dig AAAA ipv6.google.com +short
nslookup -type=AAAA google.com

# Test reverse DNS (PTR) for IPv6
dig -x 2001:4860:4860::8888 +short

# Verify /etc/resolv.conf or systemd-resolved handles IPv6 DNS
cat /etc/resolv.conf
```

## Privacy Extensions (Recommended)

```bash
# RFC 6434 recommends RFC 4941 Privacy Extensions
# Generates temporary addresses to prevent tracking

# Check if privacy extensions are enabled
cat /proc/sys/net/ipv6/conf/eth0/use_tempaddr

# Enable privacy extensions
sudo sysctl -w net.ipv6.conf.all.use_tempaddr=2
sudo sysctl -w net.ipv6.conf.default.use_tempaddr=2

# Make permanent
echo "net.ipv6.conf.all.use_tempaddr=2" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.use_tempaddr=2" | sudo tee -a /etc/sysctl.conf
```

## Compliance Checklist

```text
RFC 6434 Compliance Checklist:
[ ] Link-local address assigned on all IPv6 interfaces
[ ] Loopback address ::1 configured
[ ] ICMPv6 type 128/129 (Echo) enabled
[ ] Neighbor Discovery (NDP) operational
[ ] SLAAC working (if applicable)
[ ] DAD implemented and functional
[ ] Path MTU Discovery enabled
[ ] Packet Too Big handling working
[ ] AAAA DNS record support
[ ] Multicast Listener Discovery (MLDv2 recommended)
[ ] No blocking of mandatory ICMPv6 types
[ ] Privacy Extensions enabled (recommended)
```

RFC 6434 compliance ensures baseline interoperability across IPv6 networks, with the most common compliance gaps being firewall policies that block required ICMPv6 messages (particularly Packet Too Big for PMTU discovery) and missing DAD implementation on embedded systems.
