# How to Configure IPv6 Router Advertisements on OpenWrt

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, OpenWrt, Router Advertisement, Odhcpd, SLAAC, Networking

Description: Configure IPv6 Router Advertisements on OpenWrt using odhcpd to enable SLAAC address autoconfiguration and DNS delivery for LAN clients.

## Introduction

OpenWrt uses `odhcpd` (OpenWrt DHCP Daemon) as its default IPv6 Router Advertisement and DHCPv6 server. Unlike standalone `radvd`, `odhcpd` is tightly integrated with the UCI configuration system, making it easy to manage via both the CLI and the LuCI web interface.

## Checking odhcpd Status

```bash
# Verify odhcpd is running

/etc/init.d/odhcpd status

# Check odhcpd version
odhcpd --version
```

## Configuring RA via UCI

The `dhcp` UCI package controls odhcpd behavior:

```bash
# Configure the LAN interface for Router Advertisements + SLAAC
uci set dhcp.lan.ra=server
uci set dhcp.lan.ra_slaac=1
uci set dhcp.lan.ra_default=1
uci set dhcp.lan.ra_maxinterval=100
uci set dhcp.lan.ra_mininterval=30
uci set dhcp.lan.ra_lifetime=1800
uci commit dhcp
/etc/init.d/odhcpd restart
```

The `ra` option values:
- `server` - send RAs from this router
- `relay` - relay RAs from another router upstream
- `hybrid` - server mode when no upstream RA is available, relay otherwise
- `disabled` - no RA

## Configuring DNS via RA (RDNSS)

```bash
# Advertise custom DNS servers in RA
uci add_list dhcp.lan.dns=2001:db8:1:1::53
uci add_list dhcp.lan.dns=2606:4700:4700::1111
uci commit dhcp

# Advertise DNS search domain
uci add_list dhcp.lan.domain=example.com
uci commit dhcp

/etc/init.d/odhcpd restart
```

## Editing /etc/config/dhcp Directly

For a full view of the configuration:

```text
# /etc/config/dhcp (relevant LAN section for IPv6 RA)

config dhcp 'lan'
    option interface 'lan'
    option start '100'
    option limit '150'
    option leasetime '12h'
    # IPv6 Router Advertisement settings
    option ra 'server'
    option ra_slaac '1'
    option ra_default '1'
    option ra_maxinterval '100'
    option ra_mininterval '30'
    option ra_lifetime '1800'
    option ra_hoplimit '64'
    # DNS settings advertised in RA
    list dns '2001:db8:1:1::53'
    list domain 'example.com'
```

## Setting M/O Flags for DHCPv6 Integration

When running DHCPv6 alongside RA:

```bash
# Tell clients to use DHCPv6 for addresses (M flag)
uci set dhcp.lan.ra_management=1
# 0 = no DHCPv6, 1 = DHCPv6 for other info only (O flag)
# 2 = DHCPv6 for addresses + other info (M flag)

uci commit dhcp
/etc/init.d/odhcpd restart
```

## Configuring via LuCI (Web Interface)

1. Navigate to **Network > Interfaces > LAN > Edit**
2. Click the **IPv6 Settings** or **DHCP Server** tab
3. Under **Router Advertisement-Service**, select **Server mode**
4. Enable **Enable SLAAC**
5. Set **RA Interval** and **RA Lifetime** as desired
6. Add DNS servers under **Announced DNS servers**
7. Click **Save & Apply**

## Verifying RA on a Client

```bash
# On a client device connected to the OpenWrt LAN
# Check that a global IPv6 address was assigned via SLAAC
ip -6 addr show scope global

# Check that the default route was installed from RA
ip -6 route show default
# Expected: default via fe80::<openwrt-link-local> dev eth0 proto ra

# Verify DNS was received
cat /etc/resolv.conf
# or
systemd-resolve --status | grep "DNS Servers"
```

## Debugging odhcpd

```bash
# Enable verbose logging for odhcpd
uci set system.@system[0].log_level=7
uci commit system
/etc/init.d/log restart

# View odhcpd logs
logread | grep odhcpd

# Or run odhcpd in foreground with debug output
/etc/init.d/odhcpd stop
odhcpd -v
```

## Conclusion

OpenWrt's `odhcpd` provides a unified solution for IPv6 Router Advertisements and DHCPv6, configured entirely through the UCI system. The `ra=server` and `ra_slaac=1` options enable the most common SLAAC deployment with minimal configuration. For enterprise-style deployments requiring DHCPv6 alongside RA, adjust the `ra_management` option to set the M/O flags appropriately.
