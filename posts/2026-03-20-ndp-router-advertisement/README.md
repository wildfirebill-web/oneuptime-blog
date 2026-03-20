# How to Understand Router Advertisement (RA) Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NDP, Router Advertisement, IPv6, SLAAC, RFC 4861

Description: Understand the structure and fields of ICMPv6 Router Advertisement messages, the flags they carry, the options they include, and how hosts use them for address autoconfiguration.

## Introduction

Router Advertisement (RA) is ICMPv6 Type 134, sent by routers periodically and in response to Router Solicitations. RA messages are the primary mechanism by which hosts learn about the network: default gateway address, on-link prefixes, MTU, Hop Limit, and whether to use SLAAC or DHCPv6. Understanding every field and option in an RA is essential for configuring IPv6 correctly.

## RA Message Format

```text
ICMPv6 Router Advertisement (Type 134):

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type=134  |   Code = 0    |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Cur Hop Limit  |M|O|H|Prf|P|0|0|         Router Lifetime       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Reachable Time                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Retrans Timer                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Options ...

IPv6 Header:
  Source:      Router's link-local address (fe80::...)
  Destination: ff02::1 (all-nodes) for periodic; unicast for RS reply
  Hop Limit:   255 (mandatory)
```

## RA Fields Explained

```text
Cur Hop Limit (8 bits):
  Default Hop Limit for outgoing packets from hosts
  Typically 64; set to 0 if router doesn't specify a value
  Hosts use this value for all new outgoing packets

M flag (Managed Address Configuration):
  1 = Use DHCPv6 for address assignment (stateful)
  0 = Use SLAAC or stateless DHCPv6

O flag (Other Configuration):
  1 = Use DHCPv6 for other info (DNS, NTP) but not addresses
  0 = No DHCPv6 needed for other configuration

Router Lifetime (16 bits):
  Seconds this router is valid as a default router
  0 = Not a default router (host routes still valid)
  Typical: 1800 seconds (30 minutes)

Reachable Time (32 bits):
  Milliseconds a neighbor is considered reachable after confirmation
  0 = Not specified; hosts use their own default

Retrans Timer (32 bits):
  Milliseconds between retransmitted Neighbor Solicitations
  0 = Not specified
```

## Common RA Options

```bash
# View an actual RA being sent on your network

sudo tcpdump -i eth0 -vv "icmp6 and ip6[40] == 134"

# Example RA output with all common options:
# ICMP6, router advertisement, length 88
#   hop limit 64, Flags [managed, other stateful], pref medium,
#   router lifetime 1800s, reachable time 0ms, retrans time 0ms
#   prefix info option (3), length 32 (4): 2001:db8::/64, Flags [onlink,auto]
#     valid time 2592000s, pref. time 604800s
#   mtu option (5), length 8 (1): 1500
#   source link-address option (1), length 8 (1): 00:11:22:33:44:55
#   recursive-dns option (25), length 24 (3):
#     lifetime 1800s addr: 2001:db8::1
```

## Configuring radvd to Send RAs

```bash
# Install radvd on Linux
sudo apt-get install radvd

# /etc/radvd.conf: basic SLAAC-only configuration
sudo tee /etc/radvd.conf << 'EOF'
interface eth0 {
    AdvSendAdvert on;
    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;
    AdvDefaultLifetime 1800;
    AdvCurHopLimit 64;
    AdvManagedFlag off;      # M flag = 0: no DHCPv6 for addresses
    AdvOtherConfigFlag off;  # O flag = 0: no DHCPv6 for other config

    prefix 2001:db8::/64 {
        AdvOnLink on;
        AdvAutonomous on;    # A flag = 1: hosts may use SLAAC
        AdvValidLifetime 2592000;    # 30 days
        AdvPreferredLifetime 604800; # 7 days
    };

    # Advertise MTU option
    AdvLinkMTU 1500;

    # Advertise DNS server via RDNSS option (RFC 8106)
    RDNSS 2001:db8::1 {
        AdvRDNSSLifetime 1800;
    };
};
EOF

sudo systemctl enable radvd
sudo systemctl start radvd

# Verify radvd is sending RAs
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 134"
```

## Parsing RA Flags

```python
def parse_ra_flags(flags_byte: int) -> dict:
    """
    Parse the flags byte in an ICMPv6 Router Advertisement.
    flags_byte: The second byte of the RA body (after Cur Hop Limit)
    """
    return {
        "M_managed":     bool(flags_byte & 0x80),  # Bit 7
        "O_other":       bool(flags_byte & 0x40),  # Bit 6
        "H_home_agent":  bool(flags_byte & 0x20),  # Bit 5 (Mobile IPv6)
        "Prf_preference":((flags_byte >> 3) & 0x03),# Bits 4-3: router preference
        "P_proxy":       bool(flags_byte & 0x04),  # Bit 2 (RFC 4389)
        "address_mode": (
            "stateful DHCPv6 (M=1)"     if flags_byte & 0x80 else
            "stateless DHCPv6 (O=1)"    if flags_byte & 0x40 else
            "SLAAC only"
        )
    }

# Test cases
for flags_byte in [0x00, 0x40, 0x80, 0xC0]:
    result = parse_ra_flags(flags_byte)
    print(f"Flags=0x{flags_byte:02X}: M={result['M_managed']}, O={result['O_other']}, mode={result['address_mode']}")
```

## Conclusion

Router Advertisement is the central IPv6 autoconfiguration message. Its flags (M and O) direct hosts whether to use SLAAC, stateless DHCPv6, or stateful DHCPv6. The Prefix Information option provides the on-link prefix for SLAAC. The Router Lifetime field makes the advertising router the host's default gateway. The MTU option propagates the link MTU to hosts. Every field and option in an RA has a direct impact on how hosts configure their IPv6 addresses, routes, and parameters.
