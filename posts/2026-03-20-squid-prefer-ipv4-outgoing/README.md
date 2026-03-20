# How to Configure Squid to Prefer IPv4 Over IPv6 for Outgoing Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Squid, IPv4, IPv6, Dns_v4_first, Outgoing, Networking

Description: Configure Squid to prefer IPv4 connections to origin servers using the dns_v4_first and related directives, avoiding IPv6 connectivity issues on dual-stack hosts.

## Introduction

On dual-stack systems, Squid may default to IPv6 for outgoing connections when both A and AAAA records exist, which can cause connectivity failures if IPv6 routing is incomplete. Two directives control this: `dns_v4_first` for DNS resolution preference and `tcp_outgoing_address` for source address binding.

## Using dns_v4_first

```bash
# /etc/squid/squid.conf

# Prefer IPv4 (A records) over IPv6 (AAAA records) when resolving hostnames

dns_v4_first on

# Standard proxy configuration
http_port 10.0.0.1:3128

acl internal src 10.0.0.0/8
http_access allow internal
http_access deny all
```

## Forcing IPv4-Only DNS Resolution

For complete IPv6 avoidance, also disable IPv6 in Squid's DNS resolver:

```bash
# /etc/squid/squid.conf

# Use only IPv4 for DNS queries
dns_v4_first on

# Explicitly specify IPv4-only DNS servers
dns_nameservers 8.8.8.8 1.1.1.1

# Disable IPv6 wildcard listen address
http_port 0.0.0.0:3128   # IPv4 only (not :::3128)
```

## Verifying Outgoing Connection IP Version

```bash
# Check which IP version Squid uses for a specific host
squidclient -h 127.0.0.1 mgr:dns

# Or use cache log to see DNS resolution
sudo tail -f /var/log/squid/cache.log

# Test: request a dual-stack site and check in access log
curl -x http://10.0.0.1:3128 http://www.google.com/

# Check access log for connection details
sudo tail -5 /var/log/squid/access.log

# Use tcpdump to verify IPv4 is used for outgoing
sudo tcpdump -i eth0 host www.google.com -n | head -5
# Should show IPv4 packets (IP) not IPv6 (IP6)
```

## OS-Level IPv4 Preference

Complement Squid settings with OS-level address preference:

```bash
# /etc/gai.conf - control getaddrinfo() address selection
# Prefer IPv4 over IPv6:
precedence ::ffff:0:0/96  100  # Higher precedence for IPv4-mapped addresses

# Or set Happy Eyeballs delay to favor IPv4:
label ::1/128       0
label ::/0          1
label 2002::/16     2
label ::/96         3
label ::ffff:0:0/96 4
```

## Testing IPv4 Preference

```bash
# Verify dns_v4_first is set
sudo grep dns_v4_first /etc/squid/squid.conf

# Reload Squid
sudo squid -k reconfigure

# Test with a dual-stack host
curl -x http://10.0.0.1:3128 http://ipv4.google.com/
curl -x http://10.0.0.1:3128 http://ipv6.google.com/

# Check DNS resolution
sudo squidclient -h 127.0.0.1 mgr:all | grep -A5 dns
```

## Conclusion

Setting `dns_v4_first on` in Squid causes it to prefer A records over AAAA records when connecting to origin servers, ensuring outgoing connections use IPv4 on dual-stack systems. Combine with explicit `dns_nameservers` pointing to IPv4-only DNS servers and bind `http_port` to `0.0.0.0:3128` rather than the IPv6 wildcard for complete IPv4-only operation.
