# How to Configure Static IPv6 Addresses on FreeBSD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, FreeBSD, Static Address, Rc.conf, ifconfig

Description: Learn how to configure static IPv6 addresses on FreeBSD persistently via /etc/rc.conf and temporarily via ifconfig, including routing and DNS setup.

## Static IPv6 via /etc/rc.conf (Persistent)

```bash
# Add static IPv6 configuration to /etc/rc.conf

cat >> /etc/rc.conf << 'EOF'
# Static IPv6 address on em0
ifconfig_em0_ipv6="inet6 2001:db8::10 prefixlen 64"

# Default IPv6 gateway
ipv6_defaultrouter="2001:db8::1"
EOF

# Apply without reboot
service netif restart
service routing restart

# Verify
ifconfig em0 | grep inet6
netstat -rn -f inet6 | grep default
```

## Static IPv6 via ifconfig (Temporary)

```bash
# Add IPv6 address immediately (lost on reboot)
ifconfig em0 inet6 2001:db8::10 prefixlen 64

# Add additional IPv6 address (alias)
ifconfig em0 inet6 alias 2001:db8::20 prefixlen 64

# Add default route
route -6 add default 2001:db8::1

# Remove an IPv6 address
ifconfig em0 inet6 2001:db8::10 prefixlen 64 delete

# Remove default route
route -6 delete default
```

## Configure Multiple IPv6 Addresses

```bash
# In /etc/rc.conf for multiple addresses:
cat >> /etc/rc.conf << 'EOF'
ifconfig_em0_ipv6="inet6 2001:db8::10 prefixlen 64"
ifconfig_em0_alias0="inet6 2001:db8::20 prefixlen 64"
ifconfig_em0_alias1="inet6 fd00:db8::10 prefixlen 48"
ipv6_defaultrouter="2001:db8::1"
EOF

service netif restart
```

## Configure IPv6 DNS

```bash
# Add IPv6 DNS to /etc/resolv.conf
cat > /etc/resolv.conf << 'EOF'
# IPv6 DNS servers
nameserver 2001:4860:4860::8888
nameserver 2001:4860:4860::8844

# IPv4 fallback
nameserver 8.8.8.8

# Search domain
search example.com
EOF
```

## Full Static IPv6 Configuration Example

```bash
# /etc/rc.conf entries for a server with static IPv6
cat >> /etc/rc.conf << 'EOF'
# Network interface configuration
ifconfig_em0="inet 192.168.1.10 netmask 255.255.255.0"
ifconfig_em0_ipv6="inet6 2001:db8::10 prefixlen 64"

# Default gateways
defaultrouter="192.168.1.1"
ipv6_defaultrouter="2001:db8::1"

# Enable IPv6 (required for static IPv6 to work properly)
ipv6_activate_all_interfaces="YES"
EOF

service netif restart
service routing restart
```

## Verify Configuration

```bash
# Show IPv6 addresses
ifconfig em0 | grep inet6

# Expected output:
# inet6 fe80::1234:5678:9abc:def0%em0 prefixlen 64 scopeid 0x1
# inet6 2001:db8::10 prefixlen 64

# Check routing
netstat -rn -f inet6

# Test connectivity
ping6 -c 3 2001:db8::1        # Gateway
ping6 -c 3 2001:4860:4860::8888  # External IPv6

# Test DNS resolution
host -t AAAA google.com
```

## Summary

Configure static IPv6 on FreeBSD by adding `ifconfig_em0_ipv6="inet6 2001:db8::10 prefixlen 64"` and `ipv6_defaultrouter="2001:db8::1"` to `/etc/rc.conf`. Apply with `service netif restart && service routing restart`. For temporary changes, use `ifconfig em0 inet6 <addr> prefixlen <n>` and `route -6 add default <gw>`. Configure DNS in `/etc/resolv.conf` with `nameserver` directives.
