# How to Configure DHCPv6 Client on FreeBSD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, FreeBSD, DHCPv6, dhcp6c, Network Configuration

Description: Learn how to configure a DHCPv6 client on FreeBSD to obtain IPv6 addresses and network configuration from a DHCPv6 server, using dhcp6c.

## DHCPv6 Overview

DHCPv6 operates in two modes:
- **Stateful (IA_NA)**: Server assigns IPv6 addresses
- **Stateless**: SLAAC assigns addresses, DHCPv6 provides DNS/options only

The router's RA flags determine which mode to use:
- `M=1` → Stateful DHCPv6 for addresses
- `O=1` → Stateless DHCPv6 for options only

## Install dhcp6c (DHCPv6 Client)

```bash
# dhcp6c is part of the dhcp6 package
pkg install dhcp6

# Or check if it's already available
which dhcp6c
```

## Configure dhcp6c

```bash
# Create /usr/local/etc/dhcp6c.conf
cat > /usr/local/etc/dhcp6c.conf << 'EOF'
interface em0 {
    # Request an IPv6 address (IA_NA - Identity Association for Non-temporary Addresses)
    send ia-na 0;

    # Request a prefix delegation (IA_PD - for routers)
    # send ia-pd 0;

    # Request DNS information
    request domain-name-servers;
    request domain-name;

    # Script to run when address is assigned
    script "/usr/local/sbin/dhcp6c-run-hooks";
};

id-assoc na 0 {
    # Optional: specify preferred/valid lifetimes
};
EOF
```

## Enable dhcp6c in rc.conf

```bash
cat >> /etc/rc.conf << 'EOF'
# DHCPv6 client
dhcp6c_enable="YES"
dhcp6c_interfaces="em0"
EOF

service dhcp6c start
```

## Run dhcp6c Manually

```bash
# Start DHCPv6 client on em0 in foreground with debug
dhcp6c -d -D em0

# Start in background
dhcp6c em0

# Check if running
pgrep dhcp6c
ps aux | grep dhcp6c
```

## Verify DHCPv6 Address Assignment

```bash
# Check for DHCPv6-assigned address
ifconfig em0 | grep inet6

# DHCPv6 addresses show as 'dynamic' in some output
# Check ifconfig flags for the address

# Check dhcp6c lease file
cat /var/db/dhcp6c/dhcp6c-em0.lease

# View dhcp6c logs
grep dhcp6c /var/log/messages
```

## ISC DHCP Client (dhclient) for DHCPv6

```bash
# FreeBSD's dhclient also supports DHCPv6
# Add to rc.conf:
echo 'ifconfig_em0_ipv6="inet6 -ifdisabled"' >> /etc/rc.conf

# Some FreeBSD versions use dhclient6
# Check available tools
pkg search dhclient
```

## Stateless DHCPv6 (Options Only)

```bash
# For stateless DHCPv6 (DNS only, SLAAC for addresses):
cat > /usr/local/etc/dhcp6c.conf << 'EOF'
interface em0 {
    # Don't request addresses (SLAAC handles that)
    # Only request DNS options
    information-only;
    request domain-name-servers;
    request domain-name;
    script "/usr/local/sbin/dhcp6c-run-hooks";
};
EOF

# Also enable SLAAC for addresses
cat >> /etc/rc.conf << 'EOF'
ifconfig_em0_ipv6="inet6 accept_rtadv"
rtsold_enable="YES"
dhcp6c_enable="YES"
dhcp6c_interfaces="em0"
EOF
```

## Summary

Configure DHCPv6 client on FreeBSD with `dhcp6c`. Create `/usr/local/etc/dhcp6c.conf` with `interface em0 { send ia-na 0; request domain-name-servers; }`. Enable with `dhcp6c_enable="YES"` and `dhcp6c_interfaces="em0"` in `/etc/rc.conf`. For stateless DHCPv6 (DNS only), use `information-only` and combine with SLAAC (`accept_rtadv`). Verify with `ifconfig em0 | grep inet6` and check lease files in `/var/db/dhcp6c/`.
