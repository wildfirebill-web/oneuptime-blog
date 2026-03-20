# How to Configure a DHCPv6 Relay Agent

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, Relay Agent, IPv6, Network Architecture, Cisco, Linux

Description: Configure a DHCPv6 relay agent to forward DHCPv6 messages between clients on different subnets and a centralized DHCPv6 server, including Cisco IOS and Linux relay configurations.

## Introduction

A DHCPv6 relay agent forwards DHCPv6 messages between clients and a server when they are on different network segments. Unlike DHCPv4 which uses BOOTP relay, DHCPv6 uses dedicated relay messages (RELAY-FORW and RELAY-REPL). The relay agent is typically configured on the router or layer-3 switch that is the default gateway for the client subnet. The server does not need to be on the same link as the clients.

## DHCPv6 Relay Architecture

```yaml
DHCPv6 Relay Flow:

Client Network (2001:db8:1::/64)          Server Network
                                           (2001:db8:server::/64)

[Client]                 [Router/Relay]                [DHCPv6 Server]
   |                          |                              |
   |--SOLICIT (ff02::1:2)--->  |                              |
   |  (link-local multicast)  |                              |
   |                          |--RELAY-FORW (unicast)------> |
   |                          |  src: relay link-local       |
   |                          |  dst: 2001:db8::server       |
   |                          |  Contains original SOLICIT   |
   |                          |                              |
   |                          |<--RELAY-REPL (unicast)------ |
   |                          |  Contains ADVERTISE          |
   |                          |                              |
   |<--ADVERTISE------------- |  (relay unwraps and forwards)|
   |                          |
   [Continues: REQUEST/REPLY via relay]
```

## RELAY-FORW and RELAY-REPL Messages

```text
RELAY-FORW Message Format (Type 12):

  hop-count: number of relay agents in chain (max 32)
  link-address: global address on client-side interface
                (helps server identify which subnet client is on)
  peer-address: client's link-local address
  Options:
    Relay Message option (9): contains original client message
    Interface-ID option (18): identifies client-facing interface
    Remote-ID option (37): additional relay identification
    Subscriber-ID option (38): for subscriber management

RELAY-REPL Message Format (Type 13):
  hop-count: copied from RELAY-FORW
  link-address: copied from RELAY-FORW
  peer-address: copied from RELAY-FORW
  Options:
    Relay Message option (9): server's response (ADVERTISE/REPLY)
```

## Cisco IOS DHCPv6 Relay Configuration

```text
! Configure DHCPv6 relay on the client-facing interface

interface GigabitEthernet0/1
 description Client-facing LAN (2001:db8:1::/64)
 ipv6 address 2001:db8:1::1/64

 ! Forward DHCPv6 to server at 2001:db8:server::1
 ipv6 dhcp relay destination 2001:db8:server::1

 ! Optional: specify source address for relay messages
 ! (helps server identify which subnet)
 ipv6 dhcp relay destination 2001:db8:server::1 GigabitEthernet0/0

! Verify relay configuration
show ipv6 dhcp relay binding
show ipv6 dhcp relay statistics

! Multiple relay destinations (redundant servers):
interface GigabitEthernet0/1
 ipv6 dhcp relay destination 2001:db8:server::1
 ipv6 dhcp relay destination 2001:db8:server::2  ← secondary server

! Relay with specific link address (what server sees as client subnet):
interface GigabitEthernet0/1
 ipv6 dhcp relay destination 2001:db8:server::1
 ! link-address is automatically set to interface's global IPv6
```

## Linux dhcrelay DHCPv6 Relay

```bash
# Install ISC dhcp-relay

sudo apt-get install isc-dhcp-relay

# Configure relay
cat > /etc/default/isc-dhcp-relay << 'EOF'
# DHCPv6 relay configuration

# DHCPv6 server address
SERVERS="2001:db8:server::1"

# Interfaces to listen on (client-facing)
INTERFACES="eth1 eth2"

# Additional options
OPTIONS="-6"  # IPv6 mode
EOF

# Start relay
sudo systemctl start isc-dhcp-relay
sudo systemctl enable isc-dhcp-relay

# Manual invocation for testing:
sudo dhcrelay -6 \
    -l eth1 \                           # Listen on eth1 (client side)
    -u 2001:db8:server::1%eth0 \       # Upstream: server address on eth0
    -pf /run/dhcrelay6.pid

# Multiple client interfaces:
sudo dhcrelay -6 \
    -l eth1 -l eth2 \
    -u 2001:db8:server::1%eth0
```

## Linux dhcrelay with Interface-ID Option

```bash
# Send Interface-ID option to help server identify client subnet
# (Option 18 in RELAY-FORW)
sudo dhcrelay -6 \
    -l eth1 \
    -I \                                # Include Interface-ID option
    -u 2001:db8:server::1%eth0

# The Interface-ID helps the server:
# - Know which interface the client is on
# - Apply subnet-specific options
# - Match client subnet for address pool selection
```

## DHCPv6 Server Configuration for Relay

The DHCPv6 server needs subnet declarations for each client subnet, even if the server is not on those subnets.

```json
// Kea server: subnets for relayed clients
{
  "Dhcp6": {
    "subnet6": [
      {
        "id": 1,
        "subnet": "2001:db8:1::/64",
        "pools": [
          { "pool": "2001:db8:1::100 - 2001:db8:1::200" }
        ],
        "relay": {
          "ip-addresses": ["2001:db8:1::1"]
        }
      },
      {
        "id": 2,
        "subnet": "2001:db8:2::/64",
        "pools": [
          { "pool": "2001:db8:2::100 - 2001:db8:2::200" }
        ],
        "relay": {
          "ip-addresses": ["2001:db8:2::1"]
        }
      }
    ]
  }
}
```

## Verifying Relay Operation

```bash
# Check relay statistics on Cisco
show ipv6 dhcp relay statistics
# Should show increasing RELAY-FORW and RELAY-REPL counters

# Check relay statistics on Linux (dhcrelay)
# (dhcrelay doesn't have a statistics command; use tcpdump)

# Capture relay traffic on the server-facing interface (eth0)
sudo tcpdump -i eth0 "udp port 547" -v
# Should see RELAY-FORW (Type 12) from relay to server
# Should see RELAY-REPL (Type 13) from server to relay

# Capture client messages on client interface (eth1)
sudo tcpdump -i eth1 "udp port 546 or udp port 547" -v
# Should see SOLICIT, ADVERTISE, REQUEST, REPLY
# (relay unwraps RELAY-REPL to send ADVERTISE/REPLY to client)

# On client: verify address was received
ip -6 addr show eth0 | grep "scope global dynamic"
```

## Conclusion

DHCPv6 relay agents enable a single DHCPv6 server to serve multiple subnets. The relay intercepts client SOLICIT/REQUEST messages on the client-facing interface, wraps them in RELAY-FORW messages, and forwards them to the DHCPv6 server. The server responds with RELAY-REPL, which the relay unwraps and forwards to the client. On Cisco IOS, configure with `ipv6 dhcp relay destination`. On Linux, use `dhcrelay -6`. The DHCPv6 server must have subnet declarations for each client subnet, matched via the relay link-address.
