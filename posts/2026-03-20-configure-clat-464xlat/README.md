# How to Configure CLAT (Customer-Side Translator) for 464XLAT

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, 464XLAT, CLAT, Linux, Mobile Networks

Description: A practical guide to configuring the CLAT component of 464XLAT on Linux to enable IPv4 applications to work over an IPv6-only network connection.

## What Is CLAT?

The CLAT (Customer-side Translator) is the device-local component of 464XLAT. It creates a virtual IPv4 interface on the device, intercepts outbound IPv4 packets, translates them to IPv6, and sends them over the IPv6-only access network to the PLAT (NAT64 gateway) in the carrier/provider network.

On Android, this is built into the OS. On Linux, you can configure it manually or use tools like `clatd`.

## Prerequisites

- IPv6-only network connectivity (device has IPv6 address, no IPv4)
- A PLAT (NAT64/PLAT) in your network using prefix `64:ff9b::/96` or similar
- Linux with `ip6tables`, `iproute2`, and optionally `clatd`

## Method 1: Using clatd (Recommended)

`clatd` is a userspace CLAT daemon that handles prefix discovery and translation automatically:

```bash
# Install clatd on Ubuntu/Debian

apt install clatd

# Or install from source
git clone https://github.com/toreanderson/clatd
cd clatd
make install
```

Configure `clatd` in `/etc/clatd.conf`:

```ini
# /etc/clatd.conf

# The network interface with IPv6 connectivity (your IPv6-only uplink)
clat-dev=clat

# IPv4 address to assign to the CLAT interface
# RFC 7335 recommends 192.0.0.2/29 for CLAT
ipv4-addr=192.0.0.2

# The PLAT NAT64 prefix (auto-discovered via RFC 7050 by default)
# Uncomment to override automatic discovery:
# plat-prefix=64:ff9b::/96

# The IPv6 source address to use for translated packets
# Usually auto-configured from the uplink IPv6 address
# v6-addr=auto
```

```bash
# Start clatd
systemctl enable clatd
systemctl start clatd

# Verify CLAT interface is created
ip addr show clat
# Expected: 192.0.0.2/29

# Test IPv4 connectivity through CLAT
ping -4 8.8.8.8
```

## Method 2: Manual CLAT Configuration with TUN and Jool SIIT

For manual configuration, use Jool in SIIT mode (stateless translation):

```bash
# Load Jool SIIT kernel module
modprobe jool_siit

# Create a Jool SIIT instance for CLAT
jool_siit instance add --netfilter

# Configure the EAMT (Explicit Address Mapping Table)
# Map local IPv4 address 192.0.0.2 to device's IPv6 address
# Replace 2001:db8::device with your actual IPv6 address
jool_siit eamt add 192.0.0.2/32 2001:db8::device/128

# Set pool6 to the PLAT prefix
jool_siit pool6 add 64:ff9b::/96

# Configure iptables to intercept IPv4 packets for CLAT
iptables -t mangle -A OUTPUT -s 192.0.0.2 -j JOOL_SIIT --instance default
ip6tables -t mangle -A PREROUTING -d 2001:db8::device -j JOOL_SIIT --instance default
```

## Configuring the CLAT Interface Routing

After the CLAT interface is up, ensure IPv4 traffic routes through it:

```bash
# Verify CLAT interface is up with IPv4 address
ip addr show clat

# Add default IPv4 route through CLAT interface
# This makes all IPv4 traffic go through the CLAT translator
ip route add default dev clat

# For more specific routing, route only specific IPv4 subnets through CLAT
ip route add 0.0.0.0/0 dev clat metric 100
```

## Automatic PLAT Prefix Discovery

The CLAT discovers the PLAT's NAT64 prefix automatically using RFC 7050. This involves:

```bash
# The CLAT queries AAAA for ipv4only.arpa via the DNS64 resolver
dig AAAA ipv4only.arpa @<dns64-resolver>

# Example output when DNS64 synthesizes the record:
# ipv4only.arpa. 60 IN AAAA 64:ff9b::c000:0001
# (192.0.0.1 embedded in prefix 64:ff9b::/96)

# clatd does this automatically during startup
# Check discovered prefix in clatd logs
journalctl -u clatd | grep -i prefix
```

## Verifying CLAT Operation

```bash
# 1. Check CLAT interface has IPv4 address
ip addr show clat

# 2. Test IPv4 ping through CLAT
ping -4 -c 5 8.8.8.8

# 3. Capture to see CLAT translating IPv4 to IPv6
tcpdump -i eth0 -n 'proto 41 or ip6' &
ping -4 -c 3 8.8.8.8
fg  # and Ctrl+C

# 4. Verify no IPv4 address on the uplink interface (confirms IPv6-only)
ip addr show eth0 | grep 'inet '
```

## Testing Application Compatibility

```bash
# Test that IPv4-literal connections work through CLAT
curl -4 http://93.184.216.34/

# Test hostname-based connections
curl http://example.com

# Test an IPv4-only application
wget -4 http://8.8.8.8/
```

## Summary

CLAT is the device-side component of 464XLAT that creates a local IPv4 interface backed by IPv6 translation. The easiest way to deploy it on Linux is with `clatd`, which handles PLAT prefix discovery automatically. CLAT makes all IPv4 applications work transparently on IPv6-only networks by translating their traffic to IPv6 before it leaves the device.
