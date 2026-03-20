# How to Configure IPv6 on Aruba Wi-Fi Controllers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Aruba, Wi-Fi, Wireless Controller, AOS, SLAAC, DHCPv6

Description: Configure IPv6 support on Aruba wireless controllers and AOS, including management interface IPv6 addressing, RA forwarding to wireless clients, and IPv6 firewall policies.

---

Aruba Networks (HPE) wireless controllers running AOS support IPv6 for management interfaces, client addressing, and inter-controller communication. APs tunnel traffic back to the controller, so IPv6 configuration is centralized on the controller.

## Aruba Controller Management IPv6

```bash
# AOS CLI - Configure IPv6 on management VLAN

# Enter configuration mode
(Aruba) # configure terminal

# Configure IPv6 address on VLAN interface
(Aruba) (config) # interface vlan 1
(Aruba) (config-subif)# ipv6 address 2001:db8::controller/64
(Aruba) (config-subif)# ipv6 enable

# Set IPv6 default gateway
(Aruba) (config) # ipv6 route ::/0 2001:db8::1

# Enable IPv6 on management interface
(Aruba) (config) # interface gigabitethernet 0/0/0
(Aruba) (config-if)# ipv6 address 2001:db8::mgmt/64
(Aruba) (config-if)# ipv6 enable

# Save configuration
(Aruba) (config) # write memory
```

## Configure DHCPv6 Server on Aruba Controller

```bash
# Create IPv6 DHCP pool for wireless clients
(Aruba) (config) # ipv6 dhcp pool WIRELESS-V6
(Aruba) (config-ipv6-pool)# domain-name example.com
(Aruba) (config-ipv6-pool)# dns-server 2001:4860:4860::8888
(Aruba) (config-ipv6-pool)# dns-server 2001:4860:4860::8844
(Aruba) (config-ipv6-pool)# lease 0 1 0

# Configure stateful DHCPv6 (assign addresses)
(Aruba) (config) # ipv6 dhcp pool CORP-WIRELESS
(Aruba) (config-ipv6-pool)# address prefix 2001:db8:wifi::/64 lifetime 86400 3600

# Assign pool to interface
(Aruba) (config) # interface vlan 10
(Aruba) (config-subif)# ipv6 address 2001:db8:wifi::1/64
(Aruba) (config-subif)# ipv6 dhcp server CORP-WIRELESS
(Aruba) (config-subif)# ipv6 nd managed-config-flag
(Aruba) (config-subif)# ipv6 nd other-config-flag
```

## Configure RA on Aruba for SLAAC

```bash
# Configure Router Advertisement on VLAN interface for SLAAC
(Aruba) (config) # interface vlan 10
(Aruba) (config-subif)# ipv6 nd prefix 2001:db8:wifi::/64 \
    valid-lifetime 2592000 preferred-lifetime 604800
(Aruba) (config-subif)# ipv6 nd ra-interval 30
(Aruba) (config-subif)# ipv6 nd ra-lifetime 1800

# Stateless mode (SLAAC only, DNS via RA)
(Aruba) (config-subif)# no ipv6 nd managed-config-flag
(Aruba) (config-subif)# ipv6 nd other-config-flag
(Aruba) (config-subif)# ipv6 nd dns-server 2001:4860:4860::8888

# Verify RA configuration
(Aruba) # show ipv6 interface vlan 10
(Aruba) # show ipv6 nd interface vlan 10
```

## Aruba IPv6 Firewall Policy

```bash
# Create IPv6 firewall policy for wireless clients
(Aruba) (config) # ip access-list session V6-WIRELESS-POLICY
(Aruba) (config-sess-nacl)# ipv6 any any icmpv6 permit
(Aruba) (config-sess-nacl)# ipv6 any any udp 547 547 permit
(Aruba) (config-sess-nacl)# ipv6 any any tcp 443 any permit
(Aruba) (config-sess-nacl)# ipv6 any any tcp 80 any permit
(Aruba) (config-sess-nacl)# ipv6 any any udp 53 any permit
(Aruba) (config-sess-nacl)# ipv6 any any any deny

# Apply to user role
(Aruba) (config) # user-role wireless-employee
(Aruba) (config-role)# session-acl V6-WIRELESS-POLICY position 1
```

## Aruba ClearPass for IPv6 Client Tracking

```bash
# ClearPass Network Access Control tracks IPv6 clients

# Verify ClearPass sees IPv6 addresses
# Admin > Endpoint > Endpoints
# Filter by: IP Address contains "2001:"

# RADIUS accounting shows IPv6 attributes:
# Framed-IPv6-Address (Attribute 168)
# Framed-IPv6-Prefix (Attribute 97)

# Example RADIUS accounting log check
grep "Framed-IPv6" /var/log/radius/radius.log
```

## Aruba ArubaOS 8.x IPv6 Show Commands

```bash
# Show all IPv6 addresses on controller
show ipv6 interface brief

# Show IPv6 routing table
show ipv6 route

# Show IPv6 DHCP leases issued
show ipv6 dhcp binding

# Show IPv6 neighbors (NDP table)
show ipv6 neighbors

# Show wireless clients with IPv6
show user-table verbose | grep -A5 "IPv6"

# Show RA statistics
show ipv6 nd statistics

# Ping IPv6 from controller
ping6 2606:4700:4700::1111

# Traceroute over IPv6
traceroute6 2001:db8::gateway
```

## Aruba Instant (IAP) IPv6

```bash
# For Aruba Instant (controller-less) APs:

# Access IAP CLI
ssh admin@ap-ip-address

# Configure IPv6 on IAP VLAN
(Instant AP) # configure terminal
(Instant AP) (config) # interface vlan 1
(Instant AP) (config-vlan)# ipv6 address autoconfig

# Enable DHCPv6 relay to upstream server
(Instant AP) (config-vlan)# ipv6 dhcp-relay server-ip 2001:db8::dhcp-server

# Verify
(Instant AP) # show ipv6 interface brief
```

Aruba Wi-Fi controller IPv6 deployment requires configuring IPv6 on VLAN interfaces with RA and optionally DHCPv6, then applying IPv6-aware firewall policies to user roles. Since APs establish GRE tunnels to the controller, all client IPv6 traffic is centrally processed and policy enforced at the controller level.
