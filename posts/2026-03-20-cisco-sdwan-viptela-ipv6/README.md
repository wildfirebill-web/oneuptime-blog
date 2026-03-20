# How to Configure Cisco SD-WAN (Viptela) with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Cisco SD-WAN, Viptela, vManage, WAN, BGP, OSPF, Overlay

Description: Configure IPv6 support in Cisco SD-WAN (formerly Viptela) including service-side IPv6 addressing, transport-side IPv6, IPv6 BGP peering, and IPv6 application-aware routing policies.

---

Cisco SD-WAN supports IPv6 on both the service side (LAN-facing) and transport side (WAN-facing). IPv6 prefixes are distributed through the OMP (Overlay Management Protocol) control plane, enabling IPv6 traffic to traverse the SD-WAN fabric using IPv4 or IPv6 transport tunnels.

## Cisco SD-WAN IPv6 Architecture

```
Cisco SD-WAN IPv6 Components:
┌────────────────────────────────────────────────────────┐
│ vManage (Controller)                                   │
│ vBond (Orchestrator)                                   │
│ vSmart (Controller) - OMP distributes IPv6 routes      │
└──────────────────────────┬─────────────────────────────┘
                           │ OMP (IPv4 or IPv6 transport)
              ┌────────────┴────────────┐
         [vEdge/cEdge]            [vEdge/cEdge]
         Site A                    Site B
         Service: IPv4+IPv6        Service: IPv4+IPv6
         LAN: 2001:db8:a::/64      LAN: 2001:db8:b::/64
```

## vEdge IPv6 Service-Side Configuration

```
# vManage CLI equivalent - configure IPv6 on LAN interface

system
 host-name vEdge-SiteA
 system-ip 10.0.0.1
!
vpn 1
 name "Corporate"
 interface ge0/2
  ip address 192.168.1.1/24
  ipv6 address 2001:db8:a::1/64
  ipv6 dhcp-server
   address-pool 2001:db8:a::/64
   dns-server 2001:4860:4860::8888
  !
  no shutdown
 !
 ip route 0.0.0.0/0 vpn 0
 ipv6 route ::/0 vpn 0
!
```

## vManage Device Template for IPv6

```json
{
  "templateName": "Corporate-IPv6-Template",
  "templateDescription": "Dual-stack template with IPv6",
  "deviceType": "vedge-ISR-4331",
  "generalTemplates": [
    {
      "templateId": "vpn1-template-id",
      "templateType": "vpn-vedge",
      "subTemplates": [
        {
          "templateId": "interface-template-id",
          "templateType": "vpn-vedge-interface",
          "config": {
            "if-name": "ge0/2",
            "ip": {
              "address": "192.168.1.1/24"
            },
            "ipv6": {
              "address": "2001:db8:a::1/64",
              "dhcp-server": {
                "address-pool": "2001:db8:a::/64",
                "dns-server": ["2001:4860:4860::8888"]
              }
            }
          }
        }
      ]
    }
  ]
}
```

## OMP IPv6 Route Distribution

```
# Verify OMP distributes IPv6 prefixes
show omp routes vpn 1 family ipv6

# Output shows:
# PREFIX            FROM PEER      PREFERENCE  VPN    COLOR     STATUS
# 2001:db8:a::/64  192.168.0.1    0          1      biz-internet   C,I,R
# 2001:db8:b::/64  192.168.0.2    0          1      biz-internet   C,I,R

# Check IPv6 routes installed in VPN
show ip route vrf 1 ipv6
# or for IOS XE SD-WAN:
show ipv6 route vrf 1
```

## Cisco IOS XE SD-WAN IPv6 (cEdge)

```
! IOS XE SD-WAN device (ISR/ASR) - service VPN IPv6

! Configure service VPN interface
interface GigabitEthernet2
 vrf forwarding 1
 ip address 192.168.1.1 255.255.255.0
 ipv6 address 2001:db8:a::1/64
 ipv6 nd ra-interval 30
 ipv6 nd prefix 2001:db8:a::/64
 no ip proxy-arp
 no shutdown

! IPv6 DHCP for LAN clients
ipv6 dhcp pool VPN1-IPV6
 address prefix 2001:db8:a::/64 lifetime 86400 3600
 dns-server 2001:4860:4860::8888
 domain-name example.com

interface GigabitEthernet2
 ipv6 dhcp server VPN1-IPV6

! Verify
show ipv6 interface brief
show ipv6 dhcp binding
```

## SD-WAN IPv6 Application-Aware Routing

```
# vManage Policy for IPv6 traffic steering

# Application-Aware Routing Policy:
# Route IPv6 VoIP traffic over MPLS (low latency path)
# Route IPv6 bulk traffic over broadband

Data Policy (Centralized):
  Sequence:
    1. Match IPv6 VoIP (dscp ef, dst-port 5060/10000-20000)
       Action: set preferred-color mpls
    2. Match IPv6 Video (dscp af41)
       Action: set preferred-color mpls
    3. Default: use biz-internet

# Apply to sites
Site List: ALL-SITES
VPN List: VPN-1
```

## Verify IPv6 in SD-WAN

```bash
# On vEdge device
vEdge# show interface ge0/2 | grep ipv6
# Shows IPv6 address, state

# Verify IPv6 forwarding
vEdge# show ip route vrf 1 family ipv6
# Shows OMP-learned IPv6 routes

# Test connectivity
vEdge# ping vpn 1 2001:db8:b::10

# Check BFD session health for IPv6 transport
vEdge# show bfd sessions | grep ipv6

# On vManage dashboard:
# Monitor > Network > [Device] > Interface
# Shows IPv6 statistics per interface
```

Cisco SD-WAN IPv6 support requires configuring service-side VPN interfaces with dual-stack addresses, relying on OMP to distribute IPv6 prefixes between vEdge/cEdge devices, and optionally creating application-aware routing policies that steer IPv6 traffic based on DSCP markings or application type across the SD-WAN fabric.
