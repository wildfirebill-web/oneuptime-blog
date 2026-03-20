# How to Configure dnsmasq as an IPv6 DNS Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dnsmasq, DNS, IPv6, DHCP, SLAAC, Resolver, Local Network

Description: Configure dnsmasq to provide DNS resolution and DHCPv6/RA services over IPv6 on a local network, including AAAA record overrides and upstream forwarding.

## Introduction

dnsmasq is a lightweight DNS forwarder and DHCP server widely used on home networks, embedded devices, and local lab environments. It supports IPv6 DNS forwarding, static AAAA records, and can serve as a local DHCPv6 server.

## Installation

```bash
apt-get install -y dnsmasq
# or

dnf install -y dnsmasq
```

## Step 1: Enable IPv6 DNS Listening

```ini
# /etc/dnsmasq.conf

# Listen on specific IPv6 interface/address
listen-address=::1
listen-address=2001:db8::1

# Or listen on a specific interface
interface=eth0

# Bind only to specified addresses (prevent wildcards)
bind-interfaces

# Allow queries from IPv6 subnet
# (dnsmasq uses the interface's subnet automatically)
```

## Step 2: Upstream IPv6 Forwarders

```ini
# /etc/dnsmasq.conf

# Use IPv6 upstream resolvers
server=2606:4700:4700::1111
server=2606:4700:4700::1001
server=8.8.8.8

# Forward specific domain to internal IPv6 resolver
server=/internal.example.com/2001:db8:1::53

# No forward for local-only domains
local=/home.arpa/
local=/local/
```

## Step 3: Static AAAA Records

```ini
# /etc/dnsmasq.conf

# Static AAAA for local hostnames
address=/router.local/2001:db8::1
address=/server.local/2001:db8::10
address=/nas.local/2001:db8::20

# Or add to /etc/hosts (dnsmasq reads /etc/hosts by default)
```

```text
# /etc/hosts additions
2001:db8::1    router router.local
2001:db8::10   server server.local
```

## Step 4: DHCPv6 and RA

```ini
# /etc/dnsmasq.conf

# Enable Router Advertisements on eth0
enable-ra
dhcp-range=::1,::400,constructor:eth0,slaac,64,12h

# DHCPv6 stateful (assign from pool)
dhcp-range=2001:db8::100,2001:db8::200,64,12h

# Static DHCPv6 assignment by DUID
dhcp-host=id:00:01:00:01:12:34:56:78:aa:bb:cc:dd:ee:ff,2001:db8::50

# DNS search domain pushed to clients
dhcp-option=option6:domain-search,example.local
dhcp-option=option6:dns-server,[2001:db8::1]
```

## Step 5: DNSSEC Validation

```ini
# /etc/dnsmasq.conf

# Enable DNSSEC (requires dnsmasq compiled with DNSSEC support)
dnssec
dnssec-check-unsigned
```

```bash
# Download and configure root trust anchors
trust-anchor=.,20326,8,2,E06D44B80B8F1D39A95C0B0D7C65D08458E880409BBC683457104237C7F8EC8D
```

## Step 6: Start and Test

```bash
# Check configuration
dnsmasq --test

# Start
systemctl enable --now dnsmasq

# Verify listening
ss -lnup | grep :53

# Test
dig AAAA google.com @::1
dig AAAA router.local @::1

# Check leases
cat /var/lib/misc/dnsmasq.leases
```

## Logging

```ini
# /etc/dnsmasq.conf
log-queries
log-facility=/var/log/dnsmasq.log
log-dhcp
```

## Conclusion

dnsmasq provides simple IPv6 DNS forwarding and DHCPv6/RA in a single lightweight daemon, ideal for home labs and edge devices. Configure `listen-address` for IPv6, add static `address=` entries, and point upstream `server=` lines to IPv6 resolvers. Monitor local DNS availability with OneUptime's synthetic checks.
