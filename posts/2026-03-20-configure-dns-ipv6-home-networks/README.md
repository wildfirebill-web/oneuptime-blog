# How to Configure DNS for IPv6 on Home Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, AAAA, RDNSS, Home Network, Pi-hole

Description: Configure DNS for IPv6 on home networks including RDNSS/DHCPv6 DNS delivery, Pi-hole with IPv6, and local DNS for IPv6 hosts.

## DNS and IPv6: What's Different?

IPv6 uses `AAAA` (quad-A) DNS records instead of `A` records. The DNS resolution process is the same, but you need DNS servers that are reachable over IPv6 and can return AAAA records.

## DNS Delivery Methods in IPv6

There are two ways your home devices learn the DNS server address:

1. **RDNSS (RFC 6106)**: DNS server included in Router Advertisement messages
2. **DHCPv6 Stateless**: DNS server delivered via DHCPv6 option (when M=0, O=1)

Most modern devices support both. Configure both for maximum compatibility.

## Configuring RDNSS on Your Router

RDNSS embeds DNS server addresses in RA messages. Configure this on your router:

**OpenWRT example:**
```text
# /etc/config/network

config interface 'lan'
    option proto 'static'
    option ip6assign '64'
    list dns '2001:4860:4860::8888'
    list dns '2606:4700:4700::1111'

# /etc/config/radvd (or radvd.conf)

config interface
    option interface 'br-lan'
    list rdnss '2001:4860:4860::8888'
    list rdnss '2606:4700:4700::1111'
    option rdnss_lifetime '300'
```

## IPv6 Public DNS Servers

Use well-known IPv6-capable public DNS:

| Provider | IPv6 Addresses |
|---------|---------------|
| Google | `2001:4860:4860::8888`, `2001:4860:4860::8844` |
| Cloudflare | `2606:4700:4700::1111`, `2606:4700:4700::1001` |
| Quad9 | `2620:fe::fe`, `2620:fe::9` |
| OpenDNS | `2620:119:35::35`, `2620:119:53::53` |

## Setting Up Pi-hole for IPv6 DNS Filtering

Pi-hole can serve as a local DNS resolver with IPv6 support:

```bash
# Install Pi-hole (follow official installer)
curl -sSL https://install.pi-hole.net | bash

# During installation, provide an IPv6 DNS upstream server:
# Upstream DNS: 2001:4860:4860::8888

# After install, configure Pi-hole to listen on IPv6
# Edit /etc/pihole/setupVars.conf
IPV6_ADDRESS=2001:db8:home::2

# Restart Pi-hole DNS
pihole restartdns
```

Configure your router to advertise the Pi-hole as the DNS server:

```text
# In router RDNSS/DHCPv6 config, point to Pi-hole:
dns_server = 2001:db8:home::2   # Pi-hole's IPv6 address
```

## Local DNS for Home IPv6 Hosts

For resolving local hostnames over IPv6, use a simple DNS server like `dnsmasq`:

```text
# /etc/dnsmasq.conf

# Listen on IPv6
listen-address=::1
listen-address=2001:db8:home::1

# IPv6 AAAA records for local hosts
aaaa-record=server.home.lan,2001:db8:home::10
aaaa-record=nas.home.lan,2001:db8:home::20
aaaa-record=pi.home.lan,2001:db8:home::30

# DNS for reverse lookups (ip6.arpa)
# Reverse lookup for home prefix
rev-server=2001:db8:home::/64,127.0.0.1#5353
```

## Testing DNS Over IPv6

Verify DNS resolution is using IPv6:

```bash
# Check if DNS queries are sent over IPv6
dig AAAA google.com @2001:4860:4860::8888

# Verify local hostname resolves correctly
dig AAAA server.home.lan @2001:db8:home::1

# Check which DNS server your device is using
# Linux:
systemd-resolve --status | grep "DNS Servers"
# Mac:
scutil --dns | grep nameserver
```

## DNSSEC for IPv6 DNS

Enable DNSSEC validation for added security. Pi-hole supports this out of the box:

1. Pi-hole admin panel → Settings → DNS
2. Check "Use DNSSEC"

Or configure in Unbound (a more advanced DNS resolver):

```text
# /etc/unbound/unbound.conf
server:
    interface: ::0
    do-ip6: yes
    do-ip4: yes
    val-permissive-mode: no
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
```

## Conclusion

Configuring DNS for IPv6 home networks involves delivering DNS server addresses via RDNSS and/or DHCPv6, choosing IPv6-capable DNS resolvers, and optionally deploying Pi-hole for local filtering and DNSSEC validation. Local hostnames with AAAA records enable internal IPv6 access by name for home lab servers and devices.
