# How to Configure IPv6 Privacy Extensions on OpenWrt

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, OpenWrt, Privacy, RFC4941, Router, Networking

Description: Enable and configure IPv6 privacy extensions on OpenWrt routers to generate temporary addresses that prevent cross-network device tracking.

## Introduction

OpenWrt is a popular open-source Linux-based firmware for routers. While OpenWrt manages IPv6 addresses for its own interfaces and for clients via SLAAC, enabling privacy extensions requires configuration at both the router's own interface level and optionally instructing clients through Router Advertisements.

## Prerequisites

- OpenWrt 21.02 or later
- SSH access to the router
- Basic familiarity with the UCI configuration system

## Checking Current IPv6 Address Type

First, verify whether the router itself is using EUI-64 or privacy addresses:

```bash
# SSH into OpenWrt and check IPv6 addresses

ip -6 addr show

# Check addr_gen_mode for the WAN interface
cat /proc/sys/net/ipv6/conf/eth1/addr_gen_mode
# 0 = EUI-64, 2 = stable-privacy
```

## Enabling Privacy Extensions via UCI

OpenWrt's UCI (Unified Configuration Interface) is the primary way to configure network settings:

```bash
# Enable temporary addresses (RFC 4941) on the WAN interface
uci set network.wan.reqaddress=try
uci set network.wan.privext=2

# Apply and restart the network interface
uci commit network
/etc/init.d/network restart
```

The `privext` values correspond to the kernel's `use_tempaddr`:
- `0` = disabled
- `1` = generate but prefer stable
- `2` = generate and prefer temporary

## Configuring via /etc/config/network

Edit the network configuration file directly for more precise control:

```text
# /etc/config/network (relevant IPv6 section)

config interface 'wan6'
    option ifname 'eth1'
    option proto 'dhcpv6'
    option privext '2'
    option reqaddress 'try'
    option reqprefix 'auto'
```

After editing, apply the configuration:

```bash
# Reload network configuration without full restart
uci commit network
ifup wan6
```

## Setting addr_gen_mode for Stable Privacy

For RFC 7217 stable privacy instead of rotating temporary addresses:

```bash
# Set addr_gen_mode to stable-privacy (2) for the WAN interface
echo 2 > /proc/sys/net/ipv6/conf/eth1/addr_gen_mode

# Persist across reboots using sysctl
echo "net.ipv6.conf.eth1.addr_gen_mode = 2" >> /etc/sysctl.conf

# Apply immediately
sysctl -p /etc/sysctl.conf
```

## Configuring Privacy for LAN Clients via radvd

To encourage LAN clients to use privacy addresses, configure the Router Advertisement daemon. Note that OpenWrt uses `odhcp6c` and `radvd`:

```text
# /etc/radvd.conf
# Instruct clients to use privacy extensions

interface br-lan {
    AdvSendAdvert on;
    AdvManagedFlag off;
    AdvOtherConfigFlag off;
    prefix ::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        # Clients will handle their own privacy extensions
        # Short preferred lifetime encourages temporary address use
        AdvPreferredLifetime 14400;
        AdvValidLifetime 86400;
    };
};
```

## Verifying After Configuration

```bash
# Check active IPv6 addresses on the WAN interface
ip -6 addr show eth1

# Look for 'temporary' flag in the output indicating RFC 4941 is active
# Example: inet6 2001:db8::1a2b:3c4d:5e6f:7890/64 scope global temporary dynamic

# Verify temporary address is preferred for outbound connections
curl -6 https://ifconfig.me
```

## Applying Settings via LuCI (Web Interface)

If you prefer the web UI:

1. Navigate to **Network > Interfaces**
2. Click **Edit** on the WAN6 interface
3. Under **Advanced Settings**, look for **Privacy Extensions** or **Use temporary addresses**
4. Set to **prefer temporary addresses**
5. Click **Save & Apply**

## Conclusion

Configuring IPv6 privacy extensions on OpenWrt protects the router itself and can influence LAN client behavior. The UCI system makes it straightforward to set the `privext` option on WAN interfaces, while sysctl settings provide direct kernel-level control. After configuration, verify that the router is using a temporary or stable-privacy address for its WAN-facing IPv6 communications.
