# How to Configure IPv6 Router Advertisements on Juniper

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Juniper, Junos, Router Advertisement, SLAAC, Networking

Description: Configure IPv6 Router Advertisements on Juniper Junos routers to enable SLAAC and deliver prefix and DNS information to network clients.

## Introduction

Juniper Junos handles IPv6 Router Advertisements through the `protocols router-advertisement` stanza. Unlike Cisco where RA is configured under the interface, Juniper separates the RA configuration into its own protocol section, providing a clean separation between interface addressing and RA behavior.

## Prerequisites

- Juniper device running Junos 10.0 or later
- IPv6 forwarding enabled (default on most Junos platforms)
- Interface with IPv6 address configured

## Basic Router Advertisement Configuration

```
# Enable IPv6 Router Advertisements on ge-0/0/1 (LAN interface)

set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:1:1::1/64

set protocols router-advertisement interface ge-0/0/1.0 prefix 2001:db8:1:1::/64

# Set advertisement interval (in seconds)
set protocols router-advertisement interface ge-0/0/1.0 max-advertisement-interval 100
set protocols router-advertisement interface ge-0/0/1.0 min-advertisement-interval 30

# Router lifetime (how long this router is considered a valid default gateway)
set protocols router-advertisement interface ge-0/0/1.0 default-lifetime 1800
```

## Configuring Prefix Options

```
# Configure prefix with custom lifetimes
set protocols router-advertisement interface ge-0/0/1.0 prefix 2001:db8:1:1::/64 valid-lifetime 86400
set protocols router-advertisement interface ge-0/0/1.0 prefix 2001:db8:1:1::/64 preferred-lifetime 14400

# Disable autonomous addressing (clients use DHCPv6 for addresses)
set protocols router-advertisement interface ge-0/0/1.0 prefix 2001:db8:1:1::/64 no-autonomous-flag

# Enable on-link flag (prefix is directly reachable without routing)
set protocols router-advertisement interface ge-0/0/1.0 prefix 2001:db8:1:1::/64 onlink-flag
```

## Setting M and O Flags

```
# M flag (managed) = 1: clients should use DHCPv6 for addresses
set protocols router-advertisement interface ge-0/0/1.0 managed-configuration

# O flag (other) = 1: clients should use DHCPv6 for DNS and other config
set protocols router-advertisement interface ge-0/0/1.0 other-stateful-configuration
```

## Configuring RDNSS (DNS via RA)

Junos supports RFC 8106 RDNSS/DNSSL on modern versions:

```
# Advertise DNS server via Router Advertisement
set protocols router-advertisement interface ge-0/0/1.0 dns-server-address 2001:db8:1:1::53

# If your Junos version supports DNSSL (Junos 18.1+)
# set protocols router-advertisement interface ge-0/0/1.0 dns-search-list corp.example.com
```

## Full Configuration in Stanza Format

```
protocols {
    router-advertisement {
        interface ge-0/0/1.0 {
            max-advertisement-interval 100;
            min-advertisement-interval 30;
            default-lifetime 1800;
            prefix 2001:db8:1:1::/64 {
                valid-lifetime 86400;
                preferred-lifetime 14400;
                onlink-flag;
            }
            dns-server-address 2001:db8:1:1::53;
        }
    }
}
```

## Suppressing RA on Specific Interfaces

```
# Suppress RAs on the WAN/uplink interface
set protocols router-advertisement interface ge-0/0/0.0 no-advertisement
```

## Verifying Router Advertisement Configuration

```
# Show RA configuration
show protocols router-advertisement

# Show RA interface status and statistics
show ipv6 router-advertisement

# Show router advertisement statistics per interface
show ipv6 router-advertisement interface ge-0/0/1.0 detail

# Verify IPv6 neighbors (clients that received the RA)
show ipv6 neighbors
```

Sample output of `show ipv6 router-advertisement`:

```
Interface: ge-0/0/1.0
  Advertisement interval: 30 - 100 seconds
  Default lifetime: 1800 seconds
  Reachable time: 0 ms
  Retransmit timer: 0 ms
  Hop count: 64
  Managed config flag: No
  Other config flag: No
  Prefix: 2001:db8:1:1::/64
    Valid lifetime: 86400 seconds
    Preferred lifetime: 14400 seconds
    On-link flag: Yes
    Autonomous flag: Yes
  DNS servers: 2001:db8:1:1::53
```

## Conclusion

Juniper Junos provides clean, hierarchical Router Advertisement configuration through the `protocols router-advertisement` stanza. The separation from interface configuration makes it easy to manage RA policies independently from IP addressing. Use the prefix options to control SLAAC behavior and the DNS server configuration to deliver resolver addresses to clients without a DHCPv6 server.
