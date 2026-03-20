# How to Configure DMVPN Phase 2 with IPv4 on Cisco Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cisco, DMVPN, Phase 2, IPv4, IOS, VPN, Spoke-to-Spoke, NHRP

Description: Configure DMVPN Phase 2 on Cisco IOS to enable direct spoke-to-spoke IPv4 tunnels after initial hub routing, reducing hub bandwidth requirements in large deployments.

## Introduction

DMVPN Phase 2 extends Phase 1 by allowing spokes to build direct tunnels to each other. When Spoke A needs to reach Spoke B, it queries the NHRP hub for Spoke B's public IP, then establishes a direct IPsec/GRE tunnel — bypassing the hub for subsequent traffic.

## Key Differences from Phase 1

```
Phase 1: Spoke → Hub → Spoke (all traffic via hub)
Phase 2: Spoke → Hub (first packet) → Spoke-to-Spoke tunnel (subsequent)

Configuration changes from Phase 1:
  Hub:   Add "ip nhrp redirect" (already done in Phase 1 if following that guide)
  Spoke: Add "ip nhrp shortcut"
  Both:  EIGRP split-horizon must be disabled on hub
         Hub must NOT set next-hop-self (spoke must see real spoke next-hops)
```

## Hub Configuration (additions to Phase 1)

```cisco
interface Tunnel0
 ip nhrp redirect        ! Inform spoke of better next-hop
 ! Already configured from Phase 1:
 ! no ip split-horizon eigrp 100
 ! no ip next-hop-self eigrp 100
```

## Spoke Configuration (additions to Phase 1)

```cisco
interface Tunnel0
 ip nhrp shortcut       ! Use NHRP-resolved shortcut route

! Do NOT use default route via hub for spoke-to-spoke traffic
! Instead, use specific routes or EIGRP to learn spoke prefixes
```

## EIGRP for DMVPN Phase 2

```cisco
! Hub — EIGRP configuration
router eigrp 100
 network 10.100.0.0 0.0.0.255   ! Tunnel network
 network 192.168.1.0 0.0.0.255  ! Hub LAN

! On hub tunnel interface
interface Tunnel0
 no ip split-horizon eigrp 100
 no ip next-hop-self eigrp 100

! Spoke — EIGRP configuration
router eigrp 100
 network 10.100.0.0 0.0.0.255   ! Tunnel network
 network 192.168.2.0 0.0.0.255  ! Spoke LAN
```

## Traffic Flow in Phase 2

```
1. Spoke A sends packet to Spoke B's LAN (192.168.2.0)
2. EIGRP route shows 192.168.2.0 via 10.100.0.3 (Spoke B tunnel IP)
3. ARP/NHRP lookup: "what is the public IP of 10.100.0.3?"
4. Hub responds with NHRP redirect: "go directly to 203.0.113.3"
5. Spoke A builds direct IPsec/GRE tunnel to Spoke B's public IP
6. Subsequent packets flow Spoke A → Spoke B directly
```

## Verify Spoke-to-Spoke Tunnels

```cisco
! Show NHRP cache (includes spoke-to-spoke entries)
show ip nhrp

! Show dynamic spoke-to-spoke entries
show ip nhrp type dynamic

! Show active tunnels (should see spoke-to-spoke after traffic)
show dmvpn

! Sample output:
! Type:Spoke, NHRP Peers:2,
!  # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
!  -----  -------------   ---------------  -----  -------  -----
!      1  203.0.113.1     10.100.0.1         UP   00:05:23  S
!      1  203.0.113.3     10.100.0.3         UP   00:00:45  D   ! Dynamic spoke-to-spoke
```

## Conclusion

DMVPN Phase 2 enables direct spoke-to-spoke tunnels that form on demand. Add `ip nhrp shortcut` on spokes and ensure the hub does not advertise itself as the next-hop for spoke prefixes (use `no ip next-hop-self eigrp`). The first packet between spokes traverses the hub; NHRP resolution then creates a direct tunnel for subsequent packets.
