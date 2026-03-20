# How to Configure BGP IPv6 on Linux with FRRouting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, IPv6, FRRouting, Linux, Routing

Description: Learn how to configure BGP IPv6 unicast routing on Linux using FRRouting, including eBGP and iBGP setup with prefix advertisement and filtering.

## Overview

FRRouting provides full BGP support on Linux, including IPv6 unicast via the `address-family ipv6 unicast` configuration block in `bgpd`. This guide covers eBGP and iBGP configuration for IPv6.

## Installation

```bash
# Debian/Ubuntu
sudo apt install frr

# Enable bgpd
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons

sudo systemctl restart frr
```

## Basic eBGP IPv6 Configuration

```bash
vtysh
configure terminal

router bgp 65001
 bgp router-id 1.1.1.1

 ! eBGP neighbor using IPv6 global address
 neighbor 2001:db8:peer::2 remote-as 65002
 neighbor 2001:db8:peer::2 description "eBGP IPv6 Peer"

 ! Activate IPv6 address family for this neighbor
 address-family ipv6 unicast
  neighbor 2001:db8:peer::2 activate
  neighbor 2001:db8:peer::2 soft-reconfiguration inbound

  ! Advertise our own prefix
  network 2001:db8:1::/48

 exit-address-family

end
write memory
```

## eBGP over Link-Local Address

```bash
vtysh
configure terminal

router bgp 65001
 bgp router-id 1.1.1.1

 ! Peer using link-local address (must specify interface)
 neighbor fe80::2%eth0 remote-as 65002
 neighbor fe80::2%eth0 interface eth0    # Required for link-local peers

 address-family ipv6 unicast
  neighbor fe80::2%eth0 activate
  network 2001:db8:1::/48
 exit-address-family

end
write memory
```

## iBGP IPv6 Configuration

```bash
vtysh
configure terminal

router bgp 65001
 bgp router-id 1.1.1.1

 ! iBGP peer using IPv6 loopback address
 neighbor 2001:db8::2 remote-as 65001
 neighbor 2001:db8::2 update-source lo  ! Use loopback as source

 address-family ipv6 unicast
  neighbor 2001:db8::2 activate
  neighbor 2001:db8::2 next-hop-self   ! Replace next-hop for iBGP
 exit-address-family

end
write memory
```

## Route Filtering with Prefix Lists

```bash
vtysh
configure terminal

! Create a prefix list for inbound filtering
ipv6 prefix-list ACCEPT_PEER seq 10 permit 2001:db8:remote::/48 le 64
ipv6 prefix-list ACCEPT_PEER seq 99 deny ::/0 le 128

! Apply to BGP neighbor
router bgp 65001
 address-family ipv6 unicast
  neighbor 2001:db8:peer::2 prefix-list ACCEPT_PEER in
 exit-address-family

end
write memory
```

## Verification Commands

```bash
# Show BGP IPv6 summary
vtysh -c "show bgp ipv6 unicast summary"

# Show all IPv6 BGP routes
vtysh -c "show bgp ipv6 unicast"

# Show routes from a specific peer
vtysh -c "show bgp ipv6 unicast neighbors 2001:db8:peer::2 routes"

# Show routes advertised to a peer
vtysh -c "show bgp ipv6 unicast neighbors 2001:db8:peer::2 advertised-routes"

# Show neighbor details including capabilities
vtysh -c "show bgp neighbors 2001:db8:peer::2"
```

## Sample Output

```
Router# show bgp ipv6 unicast summary

BGP router identifier 1.1.1.1, local AS number 65001
Neighbor        V  AS     MsgRcvd  MsgSent  TblVer  InQ  OutQ  Up/Down  State/PfxRcd
2001:db8::peer  4  65002    1520     1510       5     0     0  01:12:34  4
```

## Summary

FRRouting BGP IPv6 uses `address-family ipv6 unicast` within the BGP process. Activate each neighbor in the IPv6 AF separately. Use `fe80::<addr>%<iface>` notation for link-local peers. For iBGP, use `next-hop-self` and loopback addresses. Verify with `show bgp ipv6 unicast summary` and `show bgp ipv6 unicast`.
