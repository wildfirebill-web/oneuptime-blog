# How to Configure AAAA Records in dnsmasq

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, Dnsmasq, AAAA Records, Small Network

Description: Learn how to add IPv6 AAAA records in dnsmasq for local network name resolution in home labs, small offices, and embedded Linux deployments.

## What Is dnsmasq?

dnsmasq is a lightweight DNS forwarder and DHCP server commonly used in small networks, home labs, and embedded Linux devices (routers, IoT gateways). It supports both A and AAAA records through simple host file-style configuration.

## Method 1: Using /etc/hosts (Simplest)

dnsmasq reads `/etc/hosts` by default. Adding IPv6 entries there serves both A and AAAA records:

```text
# /etc/hosts

# IPv4 entries

192.168.1.10    server1.local server1
192.168.1.20    server2.local server2

# IPv6 AAAA entries - add these for IPv6 resolution
2001:db8::10    server1.local server1
2001:db8::20    server2.local server2
```

After editing `/etc/hosts`, reload dnsmasq:

```bash
# Reload dnsmasq to pick up /etc/hosts changes
systemctl reload dnsmasq
# or send SIGHUP
kill -HUP $(cat /var/run/dnsmasq/dnsmasq.pid)
```

## Method 2: Using dnsmasq.conf host-record

For dnsmasq-native configuration without editing `/etc/hosts`, use `host-record` in `dnsmasq.conf`. This creates both A and AAAA records in a single directive:

```ini
# /etc/dnsmasq.conf

# host-record creates A and/or AAAA records simultaneously
# Format: host-record=hostname,IPv4,IPv6
host-record=server1.local,192.168.1.10,2001:db8::10
host-record=server2.local,192.168.1.20,2001:db8::20

# IPv6-only record (no IPv4) - omit the IPv4 address
host-record=ipv6only.local,,2001:db8::50

# IPv4-only record (no AAAA)
host-record=legacy.local,192.168.1.100
```

## Method 3: Using aaaa-record Directive

For explicitly adding AAAA records (without A records), use the `aaaa-record` directive:

```ini
# /etc/dnsmasq.conf

# Explicit AAAA-only record
aaaa-record=ipv6host.local,2001:db8::100

# Add AAAA alongside a separately defined A record (via hosts file or address=)
address=/server3.local/192.168.1.30
aaaa-record=server3.local,2001:db8::30
```

## Method 4: Pointing Hostnames to IPv6 Addresses with address=

The `address=` directive can set IPv6 addresses:

```ini
# Return a specific IPv6 address for all queries to a hostname
# (equivalent to a wildcard AAAA record)
address=/api.local/2001:db8::api

# Wildcard: return IPv6 for any subdomain of internal.example.com
address=/.internal.example.com/2001:db8::1
```

Note: `address=` returns the same address for all record types (A and AAAA queries both get this response). For separate A and AAAA records, use `host-record` or `/etc/hosts` instead.

## Applying Configuration Changes

```bash
# Test the dnsmasq configuration
dnsmasq --test

# Restart dnsmasq to apply changes
systemctl restart dnsmasq

# Check that dnsmasq is running
systemctl status dnsmasq
```

## Verifying AAAA Records

```bash
# Query dnsmasq for AAAA record
dig AAAA server1.local @127.0.0.1

# Expected output:
# server1.local. 0 IN AAAA 2001:db8::10

# Query for both A and AAAA
dig ANY server1.local @127.0.0.1

# Test from a client on the network
nslookup -type=AAAA server1.local 192.168.1.1
```

## Adding a Dedicated Hosts File for IPv6

For larger deployments, keep IPv6 entries in a dedicated file:

```bash
# Create a dedicated IPv6 hosts file
cat > /etc/dnsmasq-ipv6.hosts << 'EOF'
2001:db8::10 server1.local
2001:db8::20 server2.local
2001:db8::30 server3.local
EOF

# Reference it in dnsmasq.conf
echo "addn-hosts=/etc/dnsmasq-ipv6.hosts" >> /etc/dnsmasq.conf

systemctl restart dnsmasq
```

## Enabling IPv6 Support in dnsmasq

Ensure dnsmasq listens on IPv6 interfaces:

```ini
# /etc/dnsmasq.conf

# Listen on a specific IPv6 address or all interfaces
# listen-address=::1,127.0.0.1
interface=eth0

# Enable IPv6 support (usually enabled by default in modern dnsmasq)
# bind-interfaces
```

## Summary

dnsmasq supports AAAA records through `/etc/hosts` entries, the `host-record` directive (creates both A and AAAA), the `aaaa-record` directive (AAAA only), and the `address=` directive. For most small networks, adding IPv6 addresses to `/etc/hosts` alongside IPv4 entries is the simplest approach. Reload dnsmasq after changes and verify with `dig AAAA`.
