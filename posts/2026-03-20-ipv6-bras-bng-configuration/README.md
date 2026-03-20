# How to Configure IPv6 for BRAS/BNG Equipment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, BRAS, BNG, ISP, PPPoE, DHCPv6-PD, Broadband

Description: Configure IPv6 on Broadband Remote Access Server (BRAS) and Broadband Network Gateway (BNG) equipment for ISP subscriber management.

## What is a BRAS/BNG?

A BRAS (Broadband Remote Access Server) or BNG (Broadband Network Gateway) is the ISP equipment that terminates subscriber connections (PPPoE, IPoE) and applies per-subscriber policies including IPv6 prefix delegation.

## IPv6 Feature Requirements on BNG

A BNG must support:
- DHCPv6 server or relay (for prefix delegation)
- PPPoEv6 / IPoEv6 session termination
- Per-subscriber IPv6 ACLs and QoS
- RADIUS integration for IPv6 attributes
- IPv6 route injection into the core routing table

## Cisco ASR 1000 BNG Configuration

Configure IPv6 subscriber sessions on a Cisco ASR 1000:

```text
! IPv6 address pool for subscribers
ipv6 local pool SUBSCRIBER-POOL 2001:db8:subs::/40 56

! BBA group for PPPoE sessions
bba-group pppoe RESIDENTIAL
 virtual-template 1

! Virtual template with IPv6
interface Virtual-Template1
 ipv6 enable
 ipv6 address 2001:db8:bng::1/64
 peer default ipv6 pool SUBSCRIBER-POOL
 ppp authentication chap
 ppp ipcp address accept

! RADIUS server for subscriber authentication
radius server AUTH1
 address ipv6 2001:db8:radius::10 auth-port 1812 acct-port 1813
 key my-radius-secret
```

## Juniper MX BNG Configuration

On Juniper MX with Enhanced Subscriber Management:

```text
# groups {

#   subscriber-management {
set access address-assignment pool RESIDENTIAL-POOL family inet6 prefix 2001:db8:subs::/40
set access address-assignment pool RESIDENTIAL-POOL family inet6 prefix-length 56

set dynamic-profiles SUBSCRIBER-PROFILE interfaces pp0 unit "$junos-interface-unit" family inet6
set dynamic-profiles SUBSCRIBER-PROFILE interfaces pp0 unit "$junos-interface-unit" family inet6 rpf-check
set dynamic-profiles SUBSCRIBER-PROFILE interfaces pp0 unit "$junos-interface-unit" dhcpv6-server
set dynamic-profiles SUBSCRIBER-PROFILE interfaces pp0 unit "$junos-interface-unit" dhcpv6-server group RESIDENTIAL
```

## RADIUS Attributes for IPv6 Delegation

The BNG communicates with RADIUS to get per-subscriber IPv6 prefix assignments:

```text
# FreeRADIUS - return IPv6 prefix for subscriber
user@isp.com Cleartext-Password := "test123"
    Delegated-IPv6-Prefix = "2001:db8:subs:1a2b::/56",
    Framed-IPv6-Route = "2001:db8:subs:1a2b::/56 :: 1",
    Framed-IPv6-Pool = "RESIDENTIAL-POOL"
```

Key RADIUS attributes for IPv6:
- `Framed-IPv6-Address` (Attr 168): Static /128 for subscriber's WAN link
- `Delegated-IPv6-Prefix` (Attr 123): Prefix to delegate to CPE
- `Framed-IPv6-Route` (Attr 99): Static route to inject for subscriber

## Subscriber Route Injection

When a subscriber comes online, the BNG injects a host route into the routing table:

```text
! Cisco IOS - verify subscriber routes
show ipv6 route subscriber

! Expected output:
! IPv6 Routing Table - default
! S   2001:db8:subs:1a2b::/56 [1/0] via Virtual-Access1
```

## Monitoring Active Sessions

```bash
# Cisco: show active IPv6 subscriber sessions
show subscriber session all detail | include IPv6

# Juniper: show DHCP client bindings
show dhcp v6 server binding detail
```

## Conclusion

BNG IPv6 configuration involves setting up DHCPv6 prefix delegation pools, integrating with RADIUS for per-subscriber prefix assignment, and ensuring routes are properly injected for delegated prefixes. Both Cisco ASR and Juniper MX support these features natively in their broadband subscriber management modules.
