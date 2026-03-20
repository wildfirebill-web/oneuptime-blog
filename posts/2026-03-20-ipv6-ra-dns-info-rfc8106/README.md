# How to Configure IPv6 RA with DNS Information (RFC 8106)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Router Advertisement, RFC8106, RDNSS, DNSSL, DNS

Description: Configure IPv6 Router Advertisements with DNS information using RFC 8106 RDNSS and DNSSL options to deliver resolver addresses and search domains to SLAAC clients.

## Introduction

RFC 8106 (updated from RFC 6106) defines two ICMPv6 Router Advertisement options - RDNSS and DNSSL - that allow routers to deliver DNS configuration to clients without requiring a DHCPv6 server. This is essential for purely SLAAC-based IPv6 deployments.

## RFC 8106 Overview

- **RDNSS** (Recursive DNS Server Option): Contains one or more IPv6 addresses of DNS resolvers
- **DNSSL** (DNS Search List Option): Contains one or more domain names to use as DNS search suffixes
- Both options include a lifetime field, after which the information expires if no new RA is received

## Configuring RDNSS and DNSSL in radvd

```text
# /etc/radvd.conf - Full RFC 8106 configuration

interface eth1 {
    AdvSendAdvert on;
    AdvManagedFlag off;
    AdvOtherConfigFlag off;
    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;

    prefix 2001:db8:1:1::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 86400;
        AdvPreferredLifetime 14400;
    };

    # RFC 8106 RDNSS: Advertise two DNS resolvers
    # Lifetime: must be >= MaxRtrAdvInterval * 2 to avoid DNS expiry between RAs
    RDNSS 2001:db8:1:1::53 2001:4860:4860::8888 {
        AdvRDNSSLifetime 600;
    };

    # RFC 8106 DNSSL: Advertise DNS search domains
    DNSSL example.com internal.example.com {
        AdvDNSSLLifetime 600;
    };
};
```

## Lifetime Calculation

The RDNSS/DNSSL lifetime should be at least `2 * MaxRtrAdvInterval` to ensure clients do not lose DNS information between RAs:

```bash
# If MaxRtrAdvInterval = 100s:

# MinRDNSSLifetime = 2 * 100 = 200s (minimum safe)
# Recommended = 3 * 100 = 300s (safer)
# Used in examples = 600s (very safe, 6x MaxRtrAdvInterval)
```

## Verifying RDNSS/DNSSL Delivery

Use `rdisc6` to see the complete RA content:

```bash
# Send a Router Solicitation and display the full RA response
rdisc6 eth0

# Look for these sections in the output:
# Recursive DNS server : 2001:db8:1:1::53
# DNS server lifetime :           600 seconds
# DNS search list     : example.com
# Search list lifetime:           600 seconds
```

## Checking That the Client Applied the DNS

```bash
# On a Linux client with systemd-resolved:
systemd-resolve --status | grep -A 10 "Link.*eth0"
# Look for "DNS Servers" and "DNS Domain" lines

# Or check resolv.conf (on systems that write it)
cat /etc/resolv.conf

# On macOS:
scutil --dns | head -20
```

## Platform Support for RFC 8106

| Platform | RDNSS | DNSSL | Notes |
|---|---|---|---|
| Linux (kernel 3.0+) | Yes | Yes | Via rdnssd or systemd-networkd |
| Windows 10/11 | Yes | Yes | Full RFC 8106 support |
| macOS 10.7+ | Yes | Yes | Full support |
| Android 6+ | Yes | Partial | DNSSL support varies by version |
| iOS 11+ | Yes | Yes | Full support |
| FreeBSD 8+ | Yes | Yes | Via rtsold |

## Configuring on Cisco IOS-XE

```text
! IOS-XE 16.6+ supports RFC 8106 RDNSS
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 nd ra dns server 2001:db8:1:1::53 infinite
Router(config-if)# ipv6 nd ra dns server 2606:4700:4700::1111 infinite
Router(config-if)# ipv6 nd ra dns search-list corp.example.com infinite

! Verify the configuration
Router# show ipv6 interface GigabitEthernet0/0
```

## Using Infinite Lifetime

For stable DNS infrastructure that rarely changes, use an infinite (or very long) lifetime:

```text
# radvd with effectively infinite lifetime
RDNSS 2001:db8:1:1::53 {
    # 0xffffffff = infinite lifetime
    AdvRDNSSLifetime infinity;
};

DNSSL example.com {
    AdvDNSSLLifetime infinity;
};
```

Note: `infinity` is `0xffffffff` in the wire format per RFC 8106.

## Conclusion

RFC 8106 RDNSS and DNSSL options complete the stateless IPv6 client configuration picture: SLAAC provides the address, RA provides the default gateway, and RDNSS/DNSSL provide the DNS resolver and search domains. This eliminates the need for DHCPv6 in many environments. Configure the lifetime values to be at least twice the MaxRtrAdvInterval and verify delivery using `rdisc6` from a client on the same segment.
