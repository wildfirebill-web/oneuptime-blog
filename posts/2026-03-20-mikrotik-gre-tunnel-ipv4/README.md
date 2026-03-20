# How to Configure GRE Tunnel for IPv4 on MikroTik

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MikroTik, RouterOS, GRE, Tunnel, IPv4, Networking

Description: Configure a GRE tunnel between two MikroTik RouterOS devices to create a virtual point-to-point IPv4 link for routing traffic between remote networks.

## Introduction

GRE tunnels on MikroTik create virtual interfaces that encapsulate IPv4 packets within IPv4. They are useful for connecting remote sites, running routing protocols over WAN links, and creating overlay networks.

## GRE Tunnel Configuration

```text
Router A: WAN 203.0.113.1, LAN 192.168.1.0/24
Router B: WAN 203.0.113.2, LAN 192.168.2.0/24
Tunnel:   10.10.10.1/30 ←→ 10.10.10.2/30
```

## Router A

```mikrotik
# Create GRE tunnel interface

/interface gre add \
  name=gre-to-routerB \
  remote-address=203.0.113.2 \
  local-address=203.0.113.1 \
  keepalive=10s,3 \
  comment="GRE to Site B"

# Add IP to tunnel interface
/ip address add \
  address=10.10.10.1/30 \
  interface=gre-to-routerB

# Add static route to remote LAN via tunnel
/ip route add \
  dst-address=192.168.2.0/24 \
  gateway=10.10.10.2 \
  comment="Route to Site B via GRE"
```

## Router B

```mikrotik
/interface gre add \
  name=gre-to-routerA \
  remote-address=203.0.113.1 \
  local-address=203.0.113.2 \
  keepalive=10s,3

/ip address add \
  address=10.10.10.2/30 \
  interface=gre-to-routerA

/ip route add \
  dst-address=192.168.1.0/24 \
  gateway=10.10.10.1
```

## Firewall Rule to Allow GRE

```mikrotik
# Allow GRE protocol on WAN interface
/ip firewall filter add \
  chain=input \
  protocol=gre \
  action=accept \
  comment="Allow GRE from remote site" \
  place-before=0
```

## OSPF over GRE

```mikrotik
# Run OSPF across the tunnel
/routing ospf interface-template add \
  interfaces=gre-to-routerB \
  area=backbone \
  comment="OSPF over GRE"
```

## MTU Adjustment

```mikrotik
# GRE adds 24 bytes of overhead; set MTU accordingly
/interface gre set gre-to-routerB mtu=1476

# Adjust MSS clamping for TCP traffic
/ip firewall mangle add \
  chain=forward \
  protocol=tcp \
  tcp-flags=syn \
  in-interface=gre-to-routerB \
  action=change-mss \
  new-mss=1436 \
  passthrough=yes
```

## Verify GRE Tunnel

```mikrotik
# Show tunnel interface status
/interface gre print

# Show interface with stats
/interface gre print stats

# Ping across tunnel
/ping 10.10.10.2

# Test end-to-end
/ping 192.168.2.1 src-address=192.168.1.1
```

## Conclusion

MikroTik GRE tunnels are configured with `/interface gre add` specifying local and remote WAN IPs. Assign a /30 IP to the tunnel interface and add static routes for remote networks. Enable keepalive for automatic tunnel health monitoring and adjust MTU to 1476 (1500 - 24 byte GRE header) to prevent fragmentation.
