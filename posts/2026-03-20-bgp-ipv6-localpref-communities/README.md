# How to Configure BGP IPv6 Local-Pref with Communities

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, IPv6, Local-Pref, Communities, Traffic Engineering

Description: Use BGP communities to set local preference values for IPv6 routes received from upstreams and peers.

## Overview

Use BGP communities to set local preference values for IPv6 routes received from upstreams and peers.

## BGP Communities and IPv6

BGP communities are attributes attached to route announcements that carry policy signaling information. They work identically for IPv4 and IPv6 prefixes.

## Standard Community Format

Standard BGP communities (RFC 1997) are 32-bit values written as two 16-bit numbers:
```
ASN:value
65000:100    # Well-known community
65001:200    # Custom community
```

## BIRD2 Configuration Example

```
# /etc/bird/bird.conf

# Define community functions
function set_local_pref(int pref) {
    bgp_local_pref = pref;
}

# Filter for IPv6 routes with communities
filter ipv6_community_policy {
    # Honor upstream community signals for local preference
    if (65001, 100) ~ bgp_community then {
        set_local_pref(200);  # High preference
        accept;
    }
    if (65001, 200) ~ bgp_community then {
        set_local_pref(50);   # Low preference
        accept;
    }
    accept;
}

protocol bgp upstream {
    neighbor 2001:db8:peer::1 as 65001;
    ipv6 {
        import filter ipv6_community_policy;
        export filter { accept; };
    };
}
```

## FRRouting Community Configuration

```bash
# FRR vtysh configuration
router bgp 64496
  neighbor 2001:db8:peer::1 remote-as 65001
  address-family ipv6 unicast
    neighbor 2001:db8:peer::1 activate

# Route map with community matching
route-map COMMUNITY-POLICY permit 10
  match community MY-COMMUNITIES
  set local-preference 200

# Define community list
ip community-list standard MY-COMMUNITIES permit 65001:100
```

## Cisco IOS Community Configuration

```
! Configure community for IPv6 BGP
router bgp 64496
  neighbor 2001:db8:peer::1 remote-as 65001
  address-family ipv6 unicast
    neighbor 2001:db8:peer::1 route-map COMMUNITY-INBOUND in

! Route map
route-map COMMUNITY-INBOUND permit 10
  match community 100
  set local-preference 200

! Community list
ip community-list 100 permit 65001:100
```

## Testing Community Propagation

```bash
# Check if communities are present on IPv6 routes in BIRD
birdc "show route for 2001:db8::/32 all"

# In FRR
vtysh -c "show bgp ipv6 unicast 2001:db8::/32"

# Look for community attribute in output
# Example: Community: 65001:100 65001:200

# Use RIPE looking glass for external verification
curl "https://stat.ripe.net/data/bgp-state/data.json?resource=2001:db8::/32" | jq '.data.routes[].attrs.communities'
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) to monitor BGP session health for your IPv6 peers and track route counts. Unexpected drops in prefix counts may indicate community-based filtering is rejecting your routes.

## Conclusion

BGP communities work identically for IPv6 prefixes — you configure them in the same route maps and community lists. Always verify community propagation using BGP looking glasses and test policy changes in a lab environment before applying to production IPv6 BGP sessions.
