# How to Check If Your ISP Provides IPv6 - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ISP, Testing, Connectivity, DHCPv6

Description: Verify whether your ISP delivers native IPv6, including how to test from your router and devices, interpret results, and escalate when IPv6 is missing.

## Quick Test

The fastest check is to visit an IPv6 test site or ping a well-known IPv6 address.

```bash
# Ping Cloudflare's IPv6 DNS resolver

ping6 2606:4700:4700::1111

# Ping Google's IPv6 address
ping6 2001:4860:4860::8888

# Use curl to fetch an IPv6-only resource
curl -6 https://ipv6.google.com

# Check your public IPv6 address
curl -6 https://ifconfig.co
```

## Check Your Router's WAN IPv6

Log into your router's admin panel, or use CLI if your CPE runs Linux.

```bash
# On a Linux-based home router (OpenWrt, pfSense shell, etc.)

# See WAN interface IPv6 address
ip -6 addr show dev eth0.2 | grep "scope global"

# Check delegated prefix (DHCPv6-PD)
ip -6 route show | grep "::/56\|::/48\|::/64"

# Check DHCPv6 client status (dhcpcd)
dhcpcd --version
journalctl -u dhcpcd | grep -i "ipv6\|prefix\|lease" | tail -20

# Check radvd / RA from ISP
radvdump 2>/dev/null | head -30
```

## Interpret Your IPv6 Address

Identify whether you have native IPv6, a tunnel, or no IPv6.

```bash
# Check address type
ip -6 addr show | grep "scope global"

# Address types:
# 2001:xxxx::/32     - ARIN/RIPE/APNIC native allocation (good)
# 2400:xxxx::/23     - APNIC native (good)
# 2600:xxxx::/23     - ARIN native (good)
# 2a00:xxxx::/23     - RIPE NCC native (good)
# 2002:xxxx::/16     - 6to4 tunnel (legacy, often broken)
# 2001:0::/32        - Teredo tunnel (Windows legacy)
# fc00::/7           - ULA (not routable, internal only)
# fe80::/10          - Link-local only (no internet, ISP not providing IPv6)
```

## Test with Online Tools

Multiple sites provide detailed IPv6 connectivity tests.

```bash
# Test with curl (checks both IPv4 and IPv6)
curl https://test-ipv6.com/ip/?callback=x 2>/dev/null | python3 -m json.tool

# Use dig to check DNS resolves over IPv6
dig AAAA ipv6.google.com

# Test if your IPv6 DNS server is reachable
dig @2606:4700:4700::1111 AAAA cloudflare.com

# Check MTU - ISP may have issues with jumbo packets
ping6 -s 1400 2001:4860:4860::8888
ping6 -s 1452 2001:4860:4860::8888
```

## Check ISP IPv6 Support Proactively

Before signing up or calling support, verify ISP readiness.

```bash
# Check if ISP's ASN has IPv6 routes in BGP (use RIPE RIS or BGP.he.net)
# Replace 12345 with ISP's ASN
curl "https://stat.ripe.net/data/prefixes/data.json?resource=AS12345" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    print('IPv6 prefixes:', len([p for p in d['data']['prefixes'] if ':' in p['prefix']]))"

# Check if ISP's main site has AAAA record
dig AAAA isp-domain.com +short

# Check via WHOIS if ISP holds IPv6 allocation
whois -h whois.arin.net "n 198.51.100.0"   # substitute ISP IP
```

## What To Do If ISP Doesn't Provide IPv6

Options when your ISP offers no native IPv6.

```bash
# Option 1: Hurricane Electric free IPv6 tunnel broker
# Register at https://tunnelbroker.net
# Configure 6in4 tunnel on Linux:

ip tunnel add he-ipv6 mode sit remote <HE_SERVER_IP> local <YOUR_IPV4_IP> ttl 255
ip link set he-ipv6 up
ip addr add <YOUR_HE_IPV6>/64 dev he-ipv6
ip route add ::/0 dev he-ipv6
ip -6 addr show he-ipv6

# Option 2: Use OpenWrt with 6in4 package
# opkg install 6in4
# Configure via /etc/config/network

# Option 3: Wireguard VPN to an IPv6-enabled VPS
# Your VPS acts as IPv6 relay
```

## Conclusion

To verify ISP IPv6 support: ping a public IPv6 address, check your router's WAN interface for a global IPv6 address (not fe80:: or fc00::), and confirm a delegated prefix (/56 or /48) was received via DHCPv6-PD. If your ISP does not provide native IPv6, use a Hurricane Electric tunnel as a free interim solution. Most major ISPs now support IPv6; if yours does not, contact support and request IPv6 be enabled on your account - it is often available but not provisioned by default.
