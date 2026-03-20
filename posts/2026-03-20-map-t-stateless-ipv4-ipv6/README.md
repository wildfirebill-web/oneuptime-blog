# How to Implement MAP-T for Stateless IPv4 to IPv6 Translation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MAP-T, IPv6, IPv4, Stateless, Translation, ISP, Transition

Description: Configure MAP-T (Mapping of Address and Port with Translation) for stateless IPv4-to-IPv6 translation in ISP networks, enabling IPv4 connectivity over IPv6-only infrastructure without state...

## Introduction

MAP-T (RFC 7599) is a stateless IPv4-to-IPv6 translation mechanism for ISP deployments. Unlike DS-Lite (which uses tunneling), MAP-T translates IPv4 packets directly into IPv6. This eliminates per-connection state on the Border Relay (BR), enabling massive scalability. Each CE (Customer Edge) gets a deterministic port range derived algorithmically from its IPv6 address.

## How MAP-T Works

```text
IPv4 Client → CE (CPE) →[IPv4→IPv6 translation]→ IPv6 Network →[IPv6→IPv4]→ BR → IPv4 Internet
             [stateless]                                         [stateless]
```

Key elements:
- **CE (Customer Edge)**: CPE that translates IPv4 to/from IPv6. Stateless.
- **BR (Border Relay)**: ISP device that translates between IPv6 MAP-T domain and IPv4 internet. Stateless.
- **MAP domain**: Defines the IPv4 prefix, IPv6 prefix, and port sharing ratio

## MAP-T Rule Example

```text
# MAP domain parameters:

# IPv6 prefix: 2001:db8::/32 (End-user prefix length: 56)
# IPv4 prefix: 192.0.2.0/24
# EA bits: 16 (Embedded Address bits)
# PSID offset: 4 (Port Set ID)
# PSID length: 8 (256 port sets)

# A CE with IPv6 prefix 2001:db8:c012:3400::/56 maps to:
# IPv4: 192.0.2.52 (derived from EA bits)
# Port range: ports where PSID = value derived from IPv6 prefix
```

## CE Configuration on Linux

```bash
# Install MAP-T support (kernel 4.14+ with MAP-T module)
sudo modprobe ip6_tunnel

# For Linux MAP-T, use the 'map' kernel module or user-space tools
# Using iproute2 with MAP-T support:

# Create MAP-T CE interface
sudo ip link add name map0 type ip6tnl mode ip4ip6 \
  local 2001:db8:c012:3400::1 \
  remote :: \
  encaplimit none

sudo ip link set map0 up

# Configure MAP rule (requires iproute2 with MAP support or use map-t daemon)
# Rule: IPv6 prefix 2001:db8::/32, IPv4 prefix 192.0.2.0/24
# EA bits=16, PSID offset=4, PSID length=8

# Assign IPv4 address derived from MAP rule
sudo ip addr add 192.0.2.52/24 dev map0

# Route IPv4 traffic through MAP-T interface
sudo ip route add default dev map0

# IPv6 route to BR
sudo ip -6 route add 2001:db8::/32 dev eth0
```

## BR Configuration on Linux

```bash
# Border Relay handles the IPv4-internet-facing side

# Create MAP-T BR interface
sudo ip link add name br0 type ip6tnl mode ip4ip6 \
  local 2001:db8:ffff::1 \
  remote :: \
  encaplimit none

sudo ip link set br0 up

# Route MAP-T domain traffic to BR interface
sudo ip -6 route add 2001:db8::/32 dev br0

# Route IPv4 space back to internet
sudo ip route add 192.0.2.0/24 dev br0

# No NAT needed - translation is stateless
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
```

## Port Sharing (PSID)

MAP-T uses Port Set IDs to share IPv4 addresses:

```python
# Port sharing algorithm (simplified)
# Given: PSID offset=4, PSID length=8
# Port ranges for PSID=5: ports where bits 4..11 = 5

psid = 5
psid_offset = 4
psid_length = 8
ports_per_psid = 2 ** (16 - psid_offset - psid_length)  # = 16 ports per block

# Excluded ports: 0-1023 (well-known ports)
for port_start in range(1024, 65536, 2 ** (16 - psid_offset)):
    port_range = range(port_start + psid * ports_per_psid,
                       port_start + (psid + 1) * ports_per_psid)
    print(f"Port block: {port_range.start}-{port_range.stop - 1}")
```

## Verifying MAP-T

```bash
# On CE (CPE):
# Check MAP-T interface is up
ip addr show map0

# Test IPv4 connectivity
ping 8.8.8.8

# Verify source port is within allowed PSID range
# Use nc or curl and check from server side what source port was used
curl -v http://example.com 2>&1 | grep "Connected to"

# On BR:
# Monitor translations (no state, but can check counters)
ip -s link show br0

# Packet capture to verify translation
sudo tcpdump -i br0 -n 'ip or ip6'
```

## Comparison: MAP-T vs DS-Lite vs MAP-E

| Feature | MAP-T | DS-Lite | MAP-E |
|---|---|---|---|
| Mechanism | Stateless translation | Stateful tunneling | Stateless encapsulation |
| BR state | None | Per-connection | None |
| Customer NAT | Stateless (port-based) | AFTR does NAT44 | Stateless (port-based) |
| Protocol overhead | None (translation) | IPv6 header (40 bytes) | IPv6 header (40 bytes) |
| IPv4 fragmentation | Supported | Supported | Complex |

## DHCPv6 MAP-T Rule Distribution

```text
# ISP DHCPv6 server distributes MAP rules to CEs
# dhcpd6.conf
option dhcp6.map-rule code 89 = string;

subnet6 2001:db8:c000::/40 {
    range6 2001:db8:c000::1 2001:db8:cfff:ffff::1;
    # MAP-T rule option (RFC 7598)
    option dhcp6.map-rule <rule-binary>;
}
```

## Conclusion

MAP-T provides stateless IPv4-to-IPv6 translation for ISP networks. Unlike DS-Lite, there is no per-connection state on the Border Relay - each CE has a deterministic IPv4 address and port range derived algorithmically from its IPv6 prefix. This enables horizontal scaling of the BR without state synchronization. The CE translates IPv4 packets from LAN clients directly to IPv6 using the MAP-T algorithm. ISPs distribute MAP rules to CEs via DHCPv6 option 89. MAP-T is particularly suited to large-scale deployments where DS-Lite's stateful NAT becomes a bottleneck.
