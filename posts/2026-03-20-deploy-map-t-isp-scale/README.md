# How to Deploy MAP-T at ISP Scale

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MAP-T, ISP, IPv4 Transition, Stateless NAT, Translation

Description: Deploy MAP-T (Mapping of Address and Port - Translation) at ISP scale for stateless IPv4/IPv6 translation without per-session state.

## What is MAP-T?

MAP-T (RFC 7599) is a stateless IPv4/IPv6 translation mechanism. Unlike DS-Lite or NAT64, MAP-T uses algorithmic address mapping — no per-session state is needed on the ISP side, making it highly scalable.

MAP-T uses a mathematical formula to map an IPv6 subscriber prefix to a shared IPv4 address and port range. Every customer router (CE) derives its IPv4 address and allowed port range from its IPv6 prefix.

## MAP-T Parameters

- **Basic Mapping Rule (BMR)**: Defines how an IPv6 prefix maps to IPv4
- **End-User IPv6 Prefix (EUI6P)**: The /56 or /64 delegated to the customer
- **IPv4 Prefix**: The shared public IPv4 block
- **PSID (Port Set ID)**: Identifies which port range the subscriber uses

## MAP-T Rule Configuration Example

```
MAP-T Domain Parameters:
  IPv4 Prefix: 203.0.113.0/24  (256 IPv4 addresses)
  IPv6 Prefix: 2001:db8:map::/48
  PSID length: 8 bits (256 subscribers share each IPv4 address)
  Offset: 6 bits

Subscriber 1:
  IPv6 Prefix: 2001:db8:map:0001::/56
  IPv4 address: 203.0.113.1
  Port range: Ports 1024-1279 (PSID=0)
```

## Deploying MAP-T Border Relay (BR)

The BR handles translation between the MAP-T IPv6 domain and the IPv4 internet:

```bash
# Install MAP-T support via Jool (SIIT)
apt install jool-dkms jool-tools
modprobe jool_siit

# Configure SIIT instance for MAP-T translation
jool_siit instance add "mapt-br" --iptables

# Add translation pool
jool_siit -i mapt-br eamt add 2001:db8:map::/48 203.0.113.0/24
```

## CE (Customer Edge) Configuration on Linux

The CE converts private IPv4 to public IPv4+port via IPv6:

```bash
# Configure MAP-T CE - uses iproute2 tunnel
ip tunnel add mapt mode ip6tnl \
    local 2001:db8:map:0001::1 \
    remote 2001:db8:br::1 \
    mode ip4ip6

ip link set mapt up
ip route add 0.0.0.0/0 dev mapt
```

For OpenWRT CEes, MAP-T is supported via the `map` package:

```
opkg install map

# /etc/config/map
config rule
    option type 'map-t'
    option peeraddr '2001:db8:br::1'
    option ipaddr '203.0.113.0'
    option ip4prefixlen '24'
    option ip6prefix '2001:db8:map::'
    option ip6prefixlen '48'
    option ealen '8'
    option offset '6'
```

## PSID Port Assignment Verification

Verify that a subscriber's port range is correctly computed:

```python
# Calculate MAP-T port range from PSID
def calculate_map_t_ports(psid: int, psid_len: int, offset: int = 6) -> list:
    """
    Calculate allowed port range for a given PSID.
    offset: bits reserved before PSID (typically 6)
    """
    ports = []
    a_bits = 16 - offset - psid_len  # bits per port range
    for a in range(2**a_bits):
        port_start = (a << (psid_len + offset)) | (psid << offset)
        ports.append((port_start, port_start + (2**offset) - 1))
    return ports

# Subscriber with PSID=5, psid_len=8
ranges = calculate_map_t_ports(psid=5, psid_len=8)
for start, end in ranges[:5]:
    print(f"Port range: {start}-{end}")
```

## Advantages of MAP-T vs DS-Lite

| Feature | MAP-T | DS-Lite |
|---------|-------|---------|
| State on ISP | None (stateless) | Per-session NAPT state |
| Scale | Very high | Limited by AFTR capacity |
| Troubleshooting | Complex (algorithmic) | Easier (explicit mapping) |
| CPE support | Growing | Widely supported |

## Conclusion

MAP-T provides stateless IPv4/IPv6 translation for ISPs, eliminating the need for per-session NAT state at scale. The algorithmic mapping between IPv6 prefixes and IPv4 addresses makes the BR (Border Relay) simple and highly scalable. MAP-T is particularly suited for large ISPs managing thousands of concurrent sessions.
