# How to Configure DHCPv6 Relay on Juniper

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Juniper, DHCPv6, Relay, Junos, MX, EX, Networking

Description: Configure DHCPv6 relay on Juniper MX and EX series devices to forward DHCPv6 messages from client subnets to remote DHCPv6 servers.

## DHCPv6 Relay on Juniper Junos

Juniper implements DHCPv6 relay through the `dhcp-local-server` and `dhcp-relay` configuration hierarchy:

```
# Basic DHCPv6 relay — Junos MX
# Forward from clients on ge-0/0/1 to server at 2001:db8::dhcp-server

set forwarding-options helpers bootp-enable
set forwarding-options dhcp-relay group CLIENT-RELAY active-server-group DHCP-SERVERS
set forwarding-options dhcp-relay group CLIENT-RELAY interface ge-0/0/1.0

set forwarding-options dhcp-relay server-group DHCP-SERVERS 2001:db8::dhcp-server

# Enable DHCPv6 relay
set forwarding-options dhcp-relay v6 group CLIENT-RELAY
set forwarding-options dhcp-relay v6 group CLIENT-RELAY active-server-group DHCP-SERVERS
set forwarding-options dhcp-relay v6 group CLIENT-RELAY interface ge-0/0/1.0
set forwarding-options dhcp-relay v6 server-group DHCP-SERVERS 2001:db8::dhcp-server
```

## Complete DHCPv6 Relay Configuration

```
# Full Junos configuration for DHCPv6 relay

# Define DHCPv6 server group
set forwarding-options dhcp-relay v6 server-group DHCP-SERVERS 2001:db8::dhcp1
set forwarding-options dhcp-relay v6 server-group DHCP-SERVERS 2001:db8::dhcp2

# Create relay group
set forwarding-options dhcp-relay v6 group CLIENTS active-server-group DHCP-SERVERS

# Client-facing interfaces
set forwarding-options dhcp-relay v6 group CLIENTS interface ge-0/0/1.0
set forwarding-options dhcp-relay v6 group CLIENTS interface ge-0/0/2.0
set forwarding-options dhcp-relay v6 group CLIENTS interface irb.100

# Add interface ID option (Option 18) for subscriber identification
set forwarding-options dhcp-relay v6 group CLIENTS interface-id-option include
```

## DHCPv6 Relay with VRF (Routing Instances)

```
# Relay in specific routing instance (VRF)
set routing-instances Tenant1 forwarding-options dhcp-relay v6 server-group DHCP-SERVERS 2001:db8::dhcp-server
set routing-instances Tenant1 forwarding-options dhcp-relay v6 group CLIENT-RELAY active-server-group DHCP-SERVERS
set routing-instances Tenant1 forwarding-options dhcp-relay v6 group CLIENT-RELAY interface irb.100
```

## RA (Router Advertisement) Configuration for DHCPv6

```
# Configure RA flags to direct clients to use DHCPv6
# M-flag=1: use DHCPv6 for addresses
# O-flag=1: use DHCPv6 for other info only

# Managed (full DHCPv6) — M-flag
set protocols router-advertisement interface ge-0/0/1.0 managed-configuration

# Other (stateless DHCPv6) — O-flag
set protocols router-advertisement interface ge-0/0/1.0 other-stateful-configuration

# RA interval
set protocols router-advertisement interface ge-0/0/1.0 max-advertisement-interval 60
set protocols router-advertisement interface ge-0/0/1.0 min-advertisement-interval 20
```

## EX Series (Switch) DHCPv6 Relay

```
# Juniper EX switch — VLAN-based DHCPv6 relay
set vlans CLIENTS-VLAN vlan-id 100

# SVI (IRB) for client VLAN
set interfaces irb unit 100 family inet6 address 2001:db8:1::1/64

# DHCPv6 relay on IRB interface
set forwarding-options dhcp-relay v6 server-group DHCP-SERVERS 2001:db8::dhcp-server
set forwarding-options dhcp-relay v6 group VLAN-RELAY active-server-group DHCP-SERVERS
set forwarding-options dhcp-relay v6 group VLAN-RELAY interface irb.100
```

## Verification Commands

```
# Show DHCPv6 relay statistics
show dhcp v6 relay statistics

# Show relay bindings
show dhcp v6 relay binding

# Show active server groups
show dhcp v6 relay server-group

# Show relay interfaces
show dhcp v6 relay group

# Monitor relay traffic in real-time
monitor traffic interface ge-0/0/1 detail matching "udp port 547"

# System log for DHCPv6 relay
show log messages | match DHCPV6

# Clear statistics
clear dhcp v6 relay statistics all
```

## Troubleshooting

```
# Check relay is active
show dhcp v6 relay group | match Active

# Verify server reachability
ping 2001:db8::dhcp-server routing-instance default count 5

# Check for relay drops
show dhcp v6 relay statistics | match drop

# Enable DHCPv6 tracing
set system tracing destination-override syslog
set forwarding-options dhcp-relay v6 group CLIENTS overrides interface-client-limit 100
```

## Conclusion

Juniper DHCPv6 relay uses the `forwarding-options dhcp-relay v6` configuration hierarchy. Server groups can contain multiple servers for redundancy. VRF-aware relay uses per-instance `forwarding-options`. Always configure RA flags (M or O bit) on the client-facing interface to instruct clients to use DHCPv6. The `interface-id-option include` setting adds Option 18 to relay messages, providing the server with the relay interface identifier for per-subscriber policy. Use `show dhcp v6 relay statistics` to monitor message flow.
