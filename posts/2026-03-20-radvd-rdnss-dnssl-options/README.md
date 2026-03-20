# How to Configure radvd RDNSS and DNSSL Options

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Radvd, RDNSS, DNSSL, DNS, Router Advertisement

Description: Configure radvd RDNSS and DNSSL options to deliver DNS server addresses and search domain lists to IPv6 clients via Router Advertisements without requiring DHCPv6.

## Introduction

RFC 8106 defines two Router Advertisement options that allow routers to deliver DNS configuration to SLAAC clients without a DHCPv6 server:

- **RDNSS** (Recursive DNS Server): Provides the IPv6 addresses of DNS resolvers
- **DNSSL** (DNS Search List): Provides the DNS domain search suffixes

These options are supported by all modern operating systems and are essential for SLAAC-only deployments.

## Configuring RDNSS in radvd

Add `RDNSS` blocks inside the interface block:

```text
# /etc/radvd.conf

# Advertise DNS servers to clients via Router Advertisements

interface eth1 {
    AdvSendAdvert on;
    AdvManagedFlag off;
    AdvOtherConfigFlag off;

    prefix 2001:db8:1:1::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 86400;
        AdvPreferredLifetime 14400;
    };

    # Advertise primary and secondary DNS servers
    # Lifetime: how long the DNS server remains valid (seconds)
    RDNSS 2001:db8:1:1::53 2001:4860:4860::8888 {
        AdvRDNSSLifetime 600;   # 10 minutes
    };
};
```

Multiple RDNSS entries can be listed on the same line. The lifetime specifies how long the client should use these servers before discarding them if no new RA is received.

## Configuring DNSSL in radvd

Add `DNSSL` blocks to advertise search domain suffixes:

```text
interface eth1 {
    AdvSendAdvert on;

    prefix 2001:db8:1:1::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 86400;
        AdvPreferredLifetime 14400;
    };

    # Advertise RDNSS - internal resolver
    RDNSS 2001:db8:1:1::53 {
        AdvRDNSSLifetime 600;
    };

    # Advertise DNS search list - multiple domains
    DNSSL example.com internal.example.com dev.example.com {
        AdvDNSSLLifetime 600;   # 10 minutes
    };
};
```

Clients will append these suffixes when resolving unqualified hostnames. For example, `ping webserver` will try `webserver.example.com`, `webserver.internal.example.com`, etc.

## Combined RDNSS and DNSSL Example

A complete production-ready radvd configuration with DNS options:

```text
# /etc/radvd.conf - Production configuration with DNS

interface eth1 {
    AdvSendAdvert on;
    AdvManagedFlag off;
    AdvOtherConfigFlag off;
    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;
    AdvDefaultLifetime 1800;

    prefix 2001:db8:1:1::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvRouterAddr on;
        AdvValidLifetime 86400;
        AdvPreferredLifetime 14400;
    };

    # Primary and secondary DNS resolvers
    RDNSS 2001:db8:1:1::53 2001:db8:1:1::54 {
        # Lifetime should be 2-3x MaxRtrAdvInterval so DNS info doesn't expire
        # between RAs. With MaxRtrAdvInterval=100, use 200-300 seconds minimum.
        AdvRDNSSLifetime 600;
    };

    # Search domains for the local network
    DNSSL corp.example.com example.com {
        AdvDNSSLLifetime 600;
    };
};
```

## Verifying RDNSS/DNSSL on Clients

After radvd is running, verify that clients received the DNS configuration:

```bash
# On a Linux client, check the resolv.conf (if using resolvd or NetworkManager)
cat /etc/resolv.conf

# Or check the systemd-resolved state
systemd-resolve --status | grep -A5 "DNS Servers\|DNS Domain"

# Use rdisc6 to see the raw RA content including RDNSS/DNSSL
rdisc6 eth0
# Look for "Recursive DNS server" and "DNS search list" sections
```

On macOS, the DNS from RDNSS/DNSSL appears in:

```bash
# macOS: Check network service DNS settings
networksetup -getdnsservers "Wi-Fi"
scutil --dns | grep nameserver
```

## RDNSS Lifetime Guidance

Set the RDNSS/DNSSL lifetime to at least 2x the MaxRtrAdvInterval so that clients do not lose DNS configuration between RAs:

```bash
# Verify MaxRtrAdvInterval in your radvd.conf
grep MaxRtrAdvInterval /etc/radvd.conf
# If MaxRtrAdvInterval = 100s, set RDNSS lifetime >= 200s (600s is safe)
```

## Conclusion

RDNSS and DNSSL options in radvd enable fully stateless IPv6 network configuration - clients get their prefix via SLAAC and their DNS settings via RA, with no DHCPv6 server required. This simplifies network architecture and reduces dependencies. Ensure the RDNSS/DNSSL lifetime is set appropriately relative to the RA interval to prevent DNS configuration from expiring between advertisements.
