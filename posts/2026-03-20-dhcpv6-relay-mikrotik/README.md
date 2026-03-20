# How to Configure DHCPv6 Relay on MikroTik

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MikroTik, DHCPv6, Relay, RouterOS, Networking, IPv6

Description: Configure DHCPv6 relay on MikroTik RouterOS to forward DHCPv6 messages from clients to remote DHCPv6 servers.

## DHCPv6 Relay on MikroTik RouterOS

MikroTik RouterOS 7.x includes native DHCPv6 relay support:

```
# RouterOS CLI — Add DHCPv6 relay

# Step 1: Assign IPv6 address to client-facing interface
/ipv6 address
add address=2001:db8:1::1/64 interface=ether2 advertise=yes

# Step 2: Configure DHCPv6 relay
/ipv6 dhcp-relay
add name=relay-ether2 \
    interface=ether2 \
    dhcp-server=2001:db8::dhcp-server \
    local-address=2001:db8:1::1

# Verify relay configuration
/ipv6 dhcp-relay print detail
```

## Multiple Subnets with DHCPv6 Relay

```
# Relay on multiple client interfaces to the same server
/ipv6 dhcp-relay
add name=relay-vlan10 interface=vlan10 dhcp-server=2001:db8::dhcp1 local-address=2001:db8:10::1
add name=relay-vlan20 interface=vlan20 dhcp-server=2001:db8::dhcp1 local-address=2001:db8:20::1
add name=relay-vlan30 interface=vlan30 dhcp-server=2001:db8::dhcp1 local-address=2001:db8:30::1

# Print all relay configurations
/ipv6 dhcp-relay print
```

## Router Advertisement Configuration (M/O flags)

```
# Configure RA to tell clients to use DHCPv6
/ipv6 nd
set [find interface=ether2] \
    managed-address-configuration=yes \
    other-configuration=yes \
    advertise-dns=yes

# For stateless DHCPv6 (address via SLAAC, options via DHCPv6)
/ipv6 nd
set [find interface=ether2] \
    managed-address-configuration=no \
    other-configuration=yes

# Prefix advertisement
/ipv6 nd prefix
add interface=ether2 prefix=2001:db8:1::/64 autonomous=yes on-link=yes
```

## Firewall Rules for DHCPv6 Relay

```
# Allow DHCPv6 relay traffic
/ipv6 firewall filter

# Allow incoming DHCPv6 from clients (UDP 546→547)
add chain=input \
    protocol=udp dst-port=547 \
    in-interface=ether2 \
    comment="DHCPv6 from clients" \
    action=accept

# Allow outgoing relay messages to server
add chain=output \
    protocol=udp dst-port=547 \
    dst-address=2001:db8::dhcp-server \
    comment="DHCPv6 relay to server" \
    action=accept

# Allow relay replies from server
add chain=input \
    protocol=udp src-port=547 \
    src-address=2001:db8::dhcp-server \
    comment="DHCPv6 reply from server" \
    action=accept
```

## Monitoring DHCPv6 Relay

```
# Show relay status
/ipv6 dhcp-relay print detail

# Monitor DHCPv6 activity via logging
/system logging
add topics=dhcp,debug action=memory

# View DHCPv6 log
/log print where topics~"dhcp"

# Check IPv6 clients
/ipv6 dhcp-client print

# Monitor with packet sniffer (Winbox alternative: Tools > Packet Sniffer)
/tool sniffer packet print where src-mac-address="" and ip-protocol=udp
```

## Winbox / WebFig Configuration

Via Winbox GUI:
1. Navigate to **IPv6 → DHCP Relay**
2. Click **Add** (blue **+** button)
3. Set:
   - **Name**: relay-ether2
   - **Interface**: ether2 (client-facing)
   - **DHCP Server**: 2001:db8::dhcp-server
   - **Local Address**: 2001:db8:1::1
4. Click **OK**

## Troubleshooting

```
# Check relay is enabled
/ipv6 dhcp-relay print
# Should show "disabled: no"

# Enable relay if disabled
/ipv6 dhcp-relay enable [find name=relay-ether2]

# Test with ping from client VLAN
/ipv6 dhcp-server print
# If relay working, server will show bindings

# Verify DHCP server reachable
/ping address=2001:db8::dhcp-server src-address=2001:db8:1::1 count=3

# Check routing to DHCP server
/ipv6 route print where dst-address~"2001:db8::"
```

## Conclusion

MikroTik RouterOS 7.x provides simple DHCPv6 relay via the `/ipv6 dhcp-relay` menu with `interface`, `dhcp-server`, and `local-address` parameters. The local address should be the router's IPv6 address on the client-facing interface — this ensures relay messages originate from a reachable global unicast address. Always configure RA flags (`managed-address-configuration=yes`) to instruct clients to use DHCPv6. Firewall rules must explicitly allow UDP port 547 for relay traffic in both directions.
