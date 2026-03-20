# How to Understand the M Flag in Router Advertisements

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NDP, Router Advertisement, M Flag, DHCPv6, SLAAC

Description: Understand the M (Managed) flag in ICMPv6 Router Advertisements, how it directs hosts to use stateful DHCPv6 for address assignment, and how to configure it.

## Introduction

The M (Managed Address Configuration) flag in Router Advertisements is a single bit that tells IPv6 hosts whether to use DHCPv6 for address assignment. When M=1, hosts should use a stateful DHCPv6 server to obtain their IPv6 addresses. When M=0, hosts use SLAAC (Stateless Address Autoconfiguration) with prefix information from the RA. Understanding the M flag is essential for controlling how addresses are assigned in your IPv6 network.

## M Flag Semantics

```
M Flag (Managed Address Configuration):

M = 0 (default):
  Hosts SHOULD use SLAAC for address assignment
  Hosts use Prefix Information Options in the RA to form addresses
  No DHCPv6 server needed for addresses
  Appropriate for: home networks, small offices, SLAAC-only deployments

M = 1:
  Hosts SHOULD use stateful DHCPv6 for address assignment
  Hosts contact a DHCPv6 server (ff02::1:2) via Solicit message
  DHCPv6 server assigns specific addresses from a pool
  Appropriate for: enterprise networks needing address tracking,
                   networks requiring specific address assignment
```

## M Flag vs O Flag Combinations

```
M=0, O=0: Pure SLAAC
  → Addresses: from SLAAC prefix in RA
  → Other config (DNS, NTP): none via DHCPv6
  → Hosts must use RA RDNSS option or manual DNS config

M=0, O=1: SLAAC + Stateless DHCPv6
  → Addresses: from SLAAC prefix in RA
  → Other config (DNS, NTP): from stateless DHCPv6 server
  → DHCPv6 server doesn't track addresses
  → Appropriate for: networks using SLAAC but needing DHCPv6 for DNS

M=1, O=0: Stateful DHCPv6 only
  → Addresses: from DHCPv6 server
  → Other config: from DHCPv6 server (usually, unless O=0 truly)
  → Unusual: typically if M=1, O=1 together

M=1, O=1: Full DHCPv6
  → Addresses: from stateful DHCPv6 server
  → Other config: from DHCPv6 server
  → Typical enterprise deployment
```

## Configuring M Flag in radvd

```bash
# Pure SLAAC (M=0, O=0) — default
sudo tee /etc/radvd.conf << 'EOF'
interface eth0 {
    AdvSendAdvert on;
    AdvManagedFlag off;      # M = 0: SLAAC for addresses
    AdvOtherConfigFlag off;  # O = 0: no DHCPv6 for other config

    prefix 2001:db8::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 2592000;
        AdvPreferredLifetime 604800;
    };
};
EOF

# SLAAC + Stateless DHCPv6 for DNS (M=0, O=1)
sudo tee /etc/radvd.conf << 'EOF'
interface eth0 {
    AdvSendAdvert on;
    AdvManagedFlag off;      # M = 0: still use SLAAC for addresses
    AdvOtherConfigFlag on;   # O = 1: get DNS/NTP from DHCPv6

    prefix 2001:db8::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 2592000;
        AdvPreferredLifetime 604800;
    };
};
EOF

# Stateful DHCPv6 (M=1, O=1)
sudo tee /etc/radvd.conf << 'EOF'
interface eth0 {
    AdvSendAdvert on;
    AdvManagedFlag on;       # M = 1: use DHCPv6 for addresses
    AdvOtherConfigFlag on;   # O = 1: use DHCPv6 for other config

    # Prefix still present so hosts know the on-link prefix
    # But Autonomous flag is off (hosts won't use SLAAC)
    prefix 2001:db8::/64 {
        AdvOnLink on;
        AdvAutonomous off;   # A = 0: don't use SLAAC
        AdvValidLifetime 2592000;
        AdvPreferredLifetime 604800;
    };
};
EOF

sudo systemctl restart radvd
```

## Verifying M Flag with tcpdump

```bash
# Capture RA and check M flag
sudo tcpdump -i eth0 -vv "icmp6 and ip6[40] == 134"

# In the output, look for:
# "Flags [managed, other stateful]" → M=1, O=1
# "Flags [other stateful]" → M=0, O=1
# (no flags shown or "[none]") → M=0, O=0

# Check M flag byte position in RA:
# IPv6 header: 40 bytes
# ICMPv6 header: 4 bytes (Type + Code + Checksum)
# RA specific: Cur Hop Limit (1 byte), then FLAGS byte
# M flag is bit 7 (0x80) of byte at offset 45 of the IPv6+ICMPv6 data

# BPF filter: RA with M flag set
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 134 and (ip6[45] & 0x80) != 0"
```

## Checking M Flag on Hosts

```bash
# Check what M flag was received (and resulting DHCPv6 behavior)
# Linux: check if DHCPv6 client started after RA with M=1

# Monitor system log for SLAAC vs DHCPv6 decisions
journalctl -u NetworkManager | grep -E "DHCPv6|SLAAC|RA"

# Check NetworkManager connection type
nmcli connection show | grep ipv6.method
# method=auto → SLAAC (RA M=0)
# method=dhcp → DHCPv6 (RA M=1 triggered)

# Check with systemd-networkd
networkctl status eth0 | grep -E "DHCPv6|SLAAC|RA"
```

## Conclusion

The M flag in Router Advertisements is the switch between SLAAC and stateful DHCPv6 for address assignment. M=0 (SLAAC) is simpler and requires no DHCPv6 server infrastructure; M=1 (stateful DHCPv6) provides address assignment control and tracking needed in enterprise environments. The M flag does not affect DNS or NTP configuration — that is controlled by the separate O flag. When deploying DHCPv6, always set both M=1 and O=1 in the RA, and disable the Autonomous (A) flag in Prefix Information Options to prevent hosts from forming SLAAC addresses in addition to their DHCPv6 addresses.
