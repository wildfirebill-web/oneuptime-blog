# How to Configure BGP IPv6 Unicast Address Family on Juniper

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, IPv6, Juniper, Junos, Routing

Description: Learn how to configure BGP IPv6 unicast routing on Juniper routers using JunOS, including neighbor configuration, prefix advertisement, and verification.

## Overview

Juniper JunOS natively supports IPv6 BGP through the `family inet6 unicast` configuration under the BGP group or neighbor. JunOS uses a policy-based approach for route advertisements.

## Basic IPv6 BGP Configuration

```text
# Configure BGP with IPv6 address family

set protocols bgp group EBGP_V6 type external
set protocols bgp group EBGP_V6 peer-as 65002
set protocols bgp group EBGP_V6 neighbor 2001:db8:peer::2
set protocols bgp group EBGP_V6 neighbor 2001:db8:peer::2 description "eBGP IPv6 peer"
set protocols bgp group EBGP_V6 family inet6 unicast

# Set the local AS
set routing-options autonomous-system 65001
```

## Full JunOS Configuration

```text
# protocols section
protocols {
    bgp {
        group EBGP_V6 {
            type external;
            peer-as 65002;
            family inet6 {
                unicast;
            }
            neighbor 2001:db8:peer::2 {
                description "eBGP IPv6 Peer";
            }
        }
        group IBGP_V6 {
            type internal;
            local-address 2001:db8::1;    # Local loopback
            family inet6 {
                unicast;
            }
            neighbor 2001:db8::2;         # Remote iBGP peer loopback
        }
    }
}

routing-options {
    autonomous-system 65001;
}
```

## Advertising IPv6 Prefixes

Juniper uses export policies to control which routes are advertised to BGP neighbors:

```bash
# Define the prefix to advertise
set policy-options policy-statement ADVERTISE_IPV6 term own-prefixes from route-filter 2001:db8:1::/48 exact
set policy-options policy-statement ADVERTISE_IPV6 term own-prefixes then accept
set policy-options policy-statement ADVERTISE_IPV6 term default then reject

# Apply the export policy to the BGP group
set protocols bgp group EBGP_V6 export ADVERTISE_IPV6
```

## Accepting IPv6 Prefixes from Peers

```python
# Import policy - accept only specific prefixes from peers
set policy-options policy-statement ACCEPT_FROM_PEER term allowed-prefixes from family inet6
set policy-options policy-statement ACCEPT_FROM_PEER term allowed-prefixes from route-filter 2001:db8:remote::/48 orlonger
set policy-options policy-statement ACCEPT_FROM_PEER term allowed-prefixes then accept
set policy-options policy-statement ACCEPT_FROM_PEER term reject-rest then reject

set protocols bgp group EBGP_V6 import ACCEPT_FROM_PEER
```

## iBGP with Next-Hop-Self

For iBGP peers, you need next-hop-self so the next hop is reachable within the AS:

```text
# Apply next-hop-self for iBGP peers
set protocols bgp group IBGP_V6 family inet6 unicast next-hop-self
```

## Verification Commands

```text
# Show BGP IPv6 neighbor summary
show bgp neighbor

# Show all IPv6 BGP routes
show route protocol bgp table inet6.0

# Show routes advertised to a specific neighbor
show route advertising-protocol bgp 2001:db8:peer::2

# Show routes received from a specific neighbor
show route receive-protocol bgp 2001:db8:peer::2

# Detailed neighbor state including AF negotiation
show bgp neighbor 2001:db8:peer::2 | match "NLRI\|AFI\|family"
```

## Sample Output

```text
user@router> show bgp neighbor 2001:db8:peer::2

Peer: 2001:db8:peer::2 AS 65002   Local: 2001:db8:peer::1 AS 65001
  Type: External    State: Established   Flags: <>
  Last State: OpenConfirm  Last Event: RecvKeepAlive
  NLRI for restart configured on peer: inet6-unicast
  NLRI advertised by peer: inet6-unicast
  Inet6: 4 advertised, 6 received, 4 active
```

## Summary

Juniper JunOS BGP IPv6 is configured with `family inet6 unicast` in the BGP group. Use export policies to control which IPv6 prefixes are advertised and import policies to filter received routes. Verify with `show route protocol bgp table inet6.0` and `show bgp neighbor`. Always set `next-hop-self` for iBGP when the external next hop is not reachable within the AS.
