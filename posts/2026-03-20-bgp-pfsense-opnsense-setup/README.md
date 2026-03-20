# How to Set Up BGP on pfSense or OPNsense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, pfSense, OPNsense, FRR, Routing, IPv4, Networking, Firewall

Description: Learn how to configure BGP on pfSense or OPNsense using the FRR package to peer with upstream routers and advertise IPv4 prefixes.

---

pfSense and OPNsense can run FRR (Free Range Routing) to add dynamic routing capabilities including BGP. This is useful for small data centers, home labs, or ISP edge routers that need BGP peering.

## Installing FRR on pfSense

```text
System → Package Manager → Available Packages
Search: "frr" → Install "FRR"
```

On OPNsense:
```text
System → Firmware → Plugins
Search: "os-frr" → Install
```

## Accessing FRR Configuration

### pfSense
`Services → FRR Global/Zebra` (main FRR daemon)
`Services → FRR BGP` (BGP-specific config)

### OPNsense
`Services → Dynamic Routing → BGP`

## BGP Configuration via Web UI

### Global Settings

```text
FRR Global Settings:
  Enable: ✓
  Default Router ID: 10.0.0.1 (pfSense WAN IP or loopback)
  AS Number: 65001
  Log Level: informational
```

### BGP Configuration

```text
BGP Router ID: 10.0.0.1
AS Number: 65001
Networks to Advertise: 192.168.1.0/24  (your LAN subnet)
```

### Adding a BGP Neighbor

```nginx
Neighbor IP: 10.0.0.2     (upstream router's IPv4)
Remote AS: 65100          (upstream router's ASN)
Description: Upstream ISP

Update Source: WAN        (interface facing the upstream router)
```

## Raw FRR Configuration (via SSH)

For advanced settings, edit FRR configuration directly.

```bash
# Access FRR CLI

vtysh

# Show current BGP summary
show bgp summary
show ip bgp
```

```nginx
# /etc/frr/frr.conf (do not edit directly; use vtysh or the web UI)
!
frr version 8.x
frr defaults traditional
!
router bgp 65001
 bgp router-id 10.0.0.1
 !
 neighbor 10.0.0.2 remote-as 65100
 neighbor 10.0.0.2 description "Upstream Router"
 !
 address-family ipv4 unicast
  network 192.168.1.0/24
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 soft-reconfiguration inbound
 exit-address-family
!
```

## Verifying the BGP Session

```bash
# In vtysh (via SSH to pfSense/OPNsense)
vtysh -c "show bgp summary"

# Expected output when peer is established:
# Neighbor        V         AS MsgRcvd MsgSent   Up/Down  State/PfxRcd
# 10.0.0.2        4      65100      45      42 00:10:30        5

# Show received prefixes
vtysh -c "show ip bgp neighbors 10.0.0.2 received-routes"
```

## Firewall Rules for BGP

BGP uses TCP port 179. Ensure firewall rules allow BGP traffic.

```text
pfSense: Firewall → Rules → WAN
Add rule: Protocol=TCP, Destination=WAN address, Destination Port=179, Action=Pass
```

## Key Takeaways

- Install the FRR package on pfSense or OPNsense to enable BGP; configure via the web UI or `vtysh`.
- Set the Router ID to the interface IP facing the BGP peer.
- Use `network` statements to advertise your IPv4 prefixes; the prefix must exist in the routing table.
- Allow TCP port 179 in the firewall for BGP sessions to establish.
