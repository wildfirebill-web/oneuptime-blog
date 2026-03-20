# How to Configure IPv6 Privacy Extensions on pfSense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, pfSense, Privacy, RFC4941, Firewall, Networking

Description: Configure IPv6 privacy extensions on pfSense to generate temporary addresses for WAN interfaces and prevent persistent device tracking through stable EUI-64 identifiers.

## Introduction

pfSense is a widely used open-source firewall and router platform based on FreeBSD. Enabling IPv6 privacy extensions on pfSense prevents your firewall's WAN IPv6 address from being a permanent identifier derived from its hardware MAC address.

## Understanding FreeBSD IPv6 Privacy Extensions

pfSense is built on FreeBSD, which uses `net.inet6.ip6.use_tempaddr` and related sysctls to control IPv6 address generation. Unlike Linux, the settings differ slightly:

| FreeBSD sysctl | Purpose |
|---|---|
| `net.inet6.ip6.use_tempaddr` | Enable temporary address generation (RFC 4941) |
| `net.inet6.ip6.preferred_tempaddr` | Prefer temporary over permanent addresses |
| `net.inet6.ip6.temppltime` | Preferred lifetime for temporary addresses |
| `net.inet6.ip6.tempmaxlifetime` | Maximum valid lifetime for temporary addresses |

## Enabling Privacy Extensions via pfSense GUI

The simplest method is through the web interface:

1. Navigate to **Interfaces > WAN**
2. Scroll to the **IPv6 Configuration Type** section
3. If using SLAAC or DHCPv6, look for the **Use Temporary Addresses** option
4. Enable it and click **Save**, then **Apply Changes**

For SLAAC-configured interfaces:
- **Interfaces > [WAN interface] > IPv6 Configuration Type: SLAAC**
- Check the **Use Privacy Extensions** checkbox

## Enabling via the Console or SSH

For more granular control, connect via SSH or the console:

```sh
# Check current temporary address setting
sysctl net.inet6.ip6.use_tempaddr
# 0 = disabled, 1 = enabled

# Enable temporary addresses
sysctl net.inet6.ip6.use_tempaddr=1

# Prefer temporary over permanent addresses
sysctl net.inet6.ip6.preferred_tempaddr=1

# Set preferred lifetime to 24 hours (86400 seconds)
sysctl net.inet6.ip6.temppltime=86400

# Set maximum lifetime to 7 days
sysctl net.inet6.ip6.tempmaxlifetime=604800
```

## Making Settings Persistent via System Tunables

To persist these settings across reboots in pfSense:

1. Navigate to **System > Advanced > System Tunables**
2. Click **+ New** and add each sysctl:

| Tunable | Value | Description |
|---|---|---|
| `net.inet6.ip6.use_tempaddr` | `1` | Enable privacy extensions |
| `net.inet6.ip6.preferred_tempaddr` | `1` | Prefer temporary addresses |
| `net.inet6.ip6.temppltime` | `86400` | Preferred lifetime (24h) |

3. Click **Save** after adding each entry

## Verifying via Console

After applying settings, check the WAN interface for a temporary address:

```sh
# Show IPv6 addresses on the WAN interface (typically em0 or igb0)
ifconfig em0 inet6

# Look for 'temporary' keyword in the output
# Example:
# inet6 2001:db8::a3b2:c4d5:e6f7:8901 prefixlen 64 temporary autoconf
```

## Checking the Preferred Outbound Address

On pfSense, verify which IPv6 address is used for outbound connections:

```sh
# Test which source address is selected for outbound traffic
curl -6 https://ifconfig.me

# Or use fetch (FreeBSD's built-in HTTP tool)
fetch -6 -qo - https://ifconfig.me
```

The returned address should be the temporary one, not the EUI-64 or stable permanent address.

## Important Notes for pfSense

- Privacy extensions on pfSense apply to the **firewall itself**, not to LAN clients (who manage their own privacy settings)
- If you are using **static IPv6 addressing** on WAN, privacy extensions do not apply — they are only relevant for SLAAC/autoconfigured addresses
- After enabling, existing SLAAC addresses may not immediately change; consider clearing and re-acquiring the IPv6 address via **Interfaces > WAN > Release/Renew**

## Conclusion

Enabling IPv6 privacy extensions on pfSense is straightforward through either the GUI or sysctl settings. The System Tunables panel provides a persistent, reboot-safe configuration method. Once enabled, your pfSense WAN interface will use temporary, rotating IPv6 addresses that cannot be used to track your firewall across different network prefixes.
