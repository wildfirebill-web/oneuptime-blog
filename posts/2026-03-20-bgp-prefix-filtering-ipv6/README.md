# How to Configure BGP Prefix Filtering for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, IPv6, Routing Security, Filtering, Network

Description: Configure BGP prefix filters for IPv6 to prevent accepting invalid or bogon prefixes and protect your routing infrastructure.

## Why Filter BGP Prefixes?

BGP prefix filtering prevents:
- Accepting default routes from peers unintentionally
- Accepting bogon prefixes (documentation, loopback, link-local ranges)
- Route leaks from customers or peers
- Accepting overly specific routes (/128 advertisements)

## IPv6 Bogon Prefix List

These IPv6 ranges should never appear in BGP:

```
# IPv6 Bogon Prefixes (never accept from BGP peers)
::/0            # Default route (only accept if explicitly wanted)
::/8            # Unspecified address range
::1/128         # Loopback
::ffff:0:0/96   # IPv4-mapped IPv6
64:ff9b::/96    # IPv4/IPv6 translation
100::/64        # Discard prefix (RFC 6666)
2001::/32       # Teredo (RFC 4380)
2001:2::/48     # BMWG (RFC 5180)
2001:db8::/32   # Documentation (RFC 3849)
2002::/16       # 6to4 (RFC 3068)
fc00::/7        # Unique local (RFC 4193)
fe80::/10       # Link-local
fec0::/10       # Deprecated site-local
ff00::/8        # Multicast
```

## BIRD2 Prefix Filtering

```
# /etc/bird/bird.conf

# Define IPv6 bogon prefix list
define BOGON_PREFIXES_V6 = [
  ::/0,                  # Default route
  ::/8+,                 # Unspecified
  ::1/128,               # Loopback
  ::ffff:0:0/96+,        # IPv4-mapped
  2001:db8::/32+,        # Documentation
  2001:2::/48+,          # BMWG
  fc00::/7+,             # Unique local
  fe80::/10+,            # Link-local
  ff00::/8+              # Multicast
];

# Filter function for BGP imports
function ipv6_import_filter() {
    # Reject bogon prefixes
    if net ~ BOGON_PREFIXES_V6 then reject;

    # Reject too-specific prefixes (longer than /48)
    if net.len > 48 then reject;

    # Reject overly broad prefixes shorter than /19
    if net.len < 19 then reject;

    accept;
}

protocol bgp upstream_v6 {
    neighbor 2001:db8:peer::1 as 65001;
    ipv6 {
        import filter ipv6_import_filter;
        export filter { accept; };
    };
}
```

## FRRouting Prefix Lists

```
# FRR configuration
! Create IPv6 prefix-list with bogon ranges
ipv6 prefix-list BOGON-V6 seq 5 deny ::/0
ipv6 prefix-list BOGON-V6 seq 10 deny ::1/128
ipv6 prefix-list BOGON-V6 seq 15 deny 2001:db8::/32 le 128
ipv6 prefix-list BOGON-V6 seq 20 deny fc00::/7 le 128
ipv6 prefix-list BOGON-V6 seq 25 deny fe80::/10 le 128
ipv6 prefix-list BOGON-V6 seq 30 deny ff00::/8 le 128
! Deny overly specific routes
ipv6 prefix-list BOGON-V6 seq 35 deny ::/0 ge 49
! Permit everything else
ipv6 prefix-list BOGON-V6 seq 100 permit ::/0 le 48

! Apply to BGP neighbor
router bgp 64496
  neighbor 2001:db8:peer::1 remote-as 65001
  address-family ipv6 unicast
    neighbor 2001:db8:peer::1 prefix-list BOGON-V6 in
```

## Cisco IOS Prefix Filtering

```
! Define IPv6 prefix-list
ipv6 prefix-list IPV6-BOGONS seq 10 deny ::/0
ipv6 prefix-list IPV6-BOGONS seq 20 deny ::/8 le 128
ipv6 prefix-list IPV6-BOGONS seq 30 deny 2001:db8::/32 le 128
ipv6 prefix-list IPV6-BOGONS seq 40 deny fc00::/7 le 128
ipv6 prefix-list IPV6-BOGONS seq 50 deny fe80::/10 le 128
ipv6 prefix-list IPV6-BOGONS seq 60 deny ff00::/8 le 128
! Reject too-specific routes
ipv6 prefix-list IPV6-BOGONS seq 70 deny ::/0 ge 49
! Permit everything else
ipv6 prefix-list IPV6-BOGONS seq 100 permit ::/0

! Apply to BGP neighbor
router bgp 64496
  neighbor 2001:db8:peer::1 remote-as 65001
  address-family ipv6 unicast
    neighbor 2001:db8:peer::1 prefix-list IPV6-BOGONS in
```

## Using BGPq4 for Automated Filter Generation

BGPq4 generates prefix lists from IRR databases automatically:

```bash
# Install bgpq4
sudo apt-get install bgpq4

# Generate FRR prefix-list for an AS-SET
bgpq4 -6 -F "ipv6 prefix-list PEER-AS65001-IN permit %p\n" AS-EXAMPLE

# Generate Cisco format
bgpq4 -6 -Z -F "ipv6 prefix-list PEER-IN seq %n permit %p\n" AS-EXAMPLE
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your BGP sessions and track the number of accepted prefixes. Sudden drops in accepted prefixes or unexpected new prefixes in your routing table can indicate filtering misconfiguration or a route leak event.

## Conclusion

BGP prefix filtering for IPv6 is essential for routing security. Combine static bogon prefix lists with maximum-length restrictions and use BGPq4 to automate filter generation from IRR data. Combine with RPKI origin validation for defense in depth.
