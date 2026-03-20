# How to Configure DMVPN Phase 3 with IPv4 on Cisco Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cisco, DMVPN, Phase 3, IPv4, IOS, VPN, Scalability, NHRP

Description: Configure DMVPN Phase 3 on Cisco IOS to achieve hub-independent spoke-to-spoke routing using NHRP summarization, enabling hierarchical DMVPN designs.

## Introduction

DMVPN Phase 3 improves on Phase 2 by allowing the hub to advertise summarized routes while spokes still form direct tunnels. The key change is that the hub now injects NHRP routes into spokes' routing tables, allowing hierarchical DMVPN with route summarization.

## Differences Between Phases

```text
Phase 1: All traffic through hub (no spoke-to-spoke)
Phase 2: Spoke-to-spoke tunnels, but hub cannot summarize routes
Phase 3: Spoke-to-spoke tunnels + hub CAN summarize routes
         Hub uses "ip nhrp redirect" + spoke uses "ip nhrp shortcut"
         EIGRP/OSPF on hub can now advertise summaries
         Spokes install /32 NHRP routes for direct spoke access
```

## Hub Configuration

```cisco
interface Tunnel0
 description DMVPN-Phase3-Hub
 ip address 10.100.0.1 255.255.255.0
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile DMVPN-PROFILE

 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 ip nhrp authentication NHRP-KEY
 ip nhrp redirect        ! Key for Phase 3

! EIGRP with summarization on hub
router eigrp 100
 network 10.100.0.0 0.0.0.255
 network 192.168.1.0 0.0.0.255

! Hub CAN use split-horizon and next-hop-self in Phase 3
! (unlike Phase 2 where these must be disabled)
interface Tunnel0
 ip summary-address eigrp 100 10.100.0.0 255.255.255.0
```

## Spoke Configuration

```cisco
interface Tunnel0
 description DMVPN-Phase3-Spoke
 ip address 10.100.0.2 255.255.255.0
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile DMVPN-PROFILE

 ip nhrp network-id 100
 ip nhrp authentication NHRP-KEY
 ip nhrp nhs 10.100.0.1
 ip nhrp map 10.100.0.1 203.0.113.1
 ip nhrp map multicast 203.0.113.1
 ip nhrp shortcut       ! Key for Phase 3

! EIGRP on spoke
router eigrp 100
 network 10.100.0.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

## Phase 3 Traffic Flow

```text
1. Spoke A has a summary route (10.100.0.0/24) via Hub
2. Spoke A sends to Spoke B (10.100.0.3) via Hub
3. Hub receives packet, sends NHRP Redirect to Spoke A
   "Spoke B is at public IP 203.0.113.3"
4. Spoke A installs a /32 NHRP shortcut route: 10.100.0.3 → 203.0.113.3
5. Subsequent packets go Spoke A → Spoke B directly
6. NHRP shortcut expires after no traffic (default 120s)
```

## Verify Phase 3

```cisco
! Show NHRP shortcut routes installed in routing table
show ip route nhrp

! Show NHRP cache
show ip nhrp

! Show DMVPN state
show dmvpn

! Confirm spoke-to-spoke tunnel exists
show dmvpn detail | include Type|NBMA
```

## Hierarchical DMVPN

```text
Phase 3 enables hierarchical DMVPN:
  Tier 1 Hub  (regional hub)
    Tier 2 Hub  (area hub - also a spoke to Tier 1)
      Spokes  (branch offices)

Each level peers NHRP with its NHS (Next Hop Server),
and route summaries propagate upward.
```

## Conclusion

DMVPN Phase 3 is the most scalable design, allowing hub-side route summarization while still enabling direct spoke-to-spoke tunnels via NHRP shortcuts. The critical configuration additions are `ip nhrp redirect` on the hub and `ip nhrp shortcut` on spokes. Phase 3 is the recommended choice for large enterprise DMVPN deployments.
