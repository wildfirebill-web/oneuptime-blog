# How to Configure IPv6 DNS Servers on macOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, macOS, DNS, networksetup, Network Configuration

Description: Learn how to configure IPv6 DNS server addresses on macOS using networksetup, System Settings, and verify that DNS resolution works over IPv6.

## Configure IPv6 DNS via networksetup

```bash
# Set DNS servers including IPv6 addresses
# networksetup handles both IPv4 and IPv6 DNS in the same command
networksetup -setdnsservers Wi-Fi \
    2001:4860:4860::8888 \
    2001:4860:4860::8844 \
    8.8.8.8 \
    8.8.4.4

# View current DNS servers
networksetup -getdnsservers Wi-Fi

# Set DNS for Ethernet
networksetup -setdnsservers Ethernet \
    2001:4860:4860::8888 \
    2001:4860:4860::8844

# Reset DNS to DHCP/RA-provided
networksetup -setdnsservers Wi-Fi "empty"
```

## Configure DNS via System Settings

```
Steps:
1. System Settings → Network

2. Select connection → "Details..."

3. Click "DNS" tab

4. Under "DNS Servers:", click "+"

5. Add IPv6 DNS servers:
   2001:4860:4860::8888
   2001:4860:4860::8844

6. Click "OK"

Order matters: first server is primary
```

## Well-Known IPv6 DNS Servers

```
Google Public DNS:
  Primary:   2001:4860:4860::8888
  Secondary: 2001:4860:4860::8844

Cloudflare DNS:
  Primary:   2606:4700:4700::1111
  Secondary: 2606:4700:4700::1001

Quad9 DNS:
  Primary:   2620:fe::fe
  Secondary: 2620:fe::9

OpenDNS:
  Primary:   2620:119:35::35
  Secondary: 2620:119:53::53
```

## Verify DNS Configuration

```bash
# View DNS settings via scutil
scutil --dns | head -30

# Output shows resolvers in use:
# resolver #1
#   nameserver[0] : 2001:4860:4860::8888
#   nameserver[1] : 8.8.8.8

# Test AAAA record resolution
dig AAAA google.com @2001:4860:4860::8888

# Test using host command
host -t AAAA google.com 2001:4860:4860::8888

# Using nslookup
nslookup -type=AAAA google.com 2001:4860:4860::8888
```

## Test DNS Transport over IPv6

```bash
# Check if DNS query goes over IPv6
# Use tcpdump to monitor DNS traffic
sudo tcpdump -i en0 -n 'port 53 and ip6' &
dig AAAA google.com
kill %1

# If you see 'port 53' traffic with IPv6 source/destination,
# DNS is being resolved over IPv6

# Test with specific IPv6 DNS server
dig AAAA ipv6.google.com @2001:4860:4860::8888 +stats
```

## Configure DNS Search Domains

```bash
# Set search domains
networksetup -setsearchdomains Wi-Fi example.com corp.example.com

# View current search domains
networksetup -getsearchdomains Wi-Fi

# Clear search domains
networksetup -setsearchdomains Wi-Fi "empty"
```

## Flush DNS Cache

```bash
# Flush macOS DNS cache
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# Verify cache is flushed (check DNS resolution timing)
time dig A google.com   # First query hits DNS server
time dig A google.com   # Second query may be cached
```

## Summary

Configure IPv6 DNS on macOS with `networksetup -setdnsservers Wi-Fi 2001:4860:4860::8888 8.8.8.8` or via **System Settings → Network → DNS tab**. Both IPv4 and IPv6 DNS servers are configured together. View current DNS with `scutil --dns`. Verify with `dig AAAA google.com @2001:4860:4860::8888`. Flush the DNS cache with `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder`.
