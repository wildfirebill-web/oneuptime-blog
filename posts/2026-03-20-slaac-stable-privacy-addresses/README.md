# How to Understand Stable Privacy Addresses (RFC 7217)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Stable Privacy, SLAAC, IPv6, RFC 7217, Interface Identifier, Privacy

Description: Understand how RFC 7217 stable privacy addresses generate a unique but stable interface identifier per network, providing privacy without the address instability of random temporary addresses.

## Introduction

RFC 7217 "Stable and Opaque IIDs with IPv6 SLAAC" defines an algorithm for generating interface identifiers that are stable within a network (the same address is used every time you connect to the same network) but different across networks (different address for each SSID/network). Unlike EUI-64 (stable but trackable across networks) and privacy extensions (random but unstable), RFC 7217 addresses are pseudo-random but deterministic per network, combining the benefits of both approaches.

## Comparison of Interface Identifier Methods

```
Interface Identifier Method Comparison:

Method              | Stable per network | Privacy (no MAC) | Changes over time
EUI-64 (MAC-based)  | Yes (global)       | No               | Never
Privacy Ext (RFC 8981)| No (random each time)| Yes          | Yes (daily)
Stable Privacy (RFC 7217)| Yes (per-network) | Yes          | Only on network change

EUI-64:
  Same IID on all networks: 0211:22ff:fe33:4455
  MAC is visible in address: traceable across networks
  Never changes: persistent identity

Privacy Extensions:
  Random IID changes daily: hard to track over time
  But: existing connections break when address rotates
  Not suitable for servers or stable applications

RFC 7217 Stable Privacy:
  IID = hash(prefix + network_id + interface + secret_key + DAD_counter)
  Same network: same address (stable sessions)
  Different network: different address (privacy)
  Never reveals MAC address
  Does not change unless network changes
```

## RFC 7217 Algorithm

```
Stable Privacy IID Generation:

IID = F(Prefix, Net_Iface, Network_ID, DAD_Counter, secret_key)

Where F is a cryptographic hash (e.g., SHA-256)

Parameters:
  Prefix:       Network prefix from RA (e.g., 2001:db8::/64)
  Net_Iface:    Interface identifier (e.g., "eth0" or interface index)
  Network_ID:   Network identifier (e.g., SSID for Wi-Fi, or empty)
  DAD_Counter:  Starts at 0, incremented if DAD detects collision
  secret_key:   Host-specific random key (generated once, stored)

Properties:
  1. Deterministic: same inputs → same IID
  2. Stable: same network → same IID across reboots
  3. Private: different networks → different IIDs
  4. No MAC: IID does not reveal MAC address
  5. Collision-resistant: DAD_Counter handles collisions

Example outputs (all from MAC 00:11:22:33:44:55):
  Home network (2001:db8:a::/64):   2001:db8:a::9c23:5f1a:b2e4:7d30
  Work network (2001:db8:b::/64):   2001:db8:b::3f91:28ac:e654:102b
  Coffee shop (2001:db8:c::/64):    2001:db8:c::7b4e:91f2:3a58:cd67

Addresses look random but:
  Next reboot at home:  2001:db8:a::9c23:5f1a:b2e4:7d30 (same!)
  Next reboot at work:  2001:db8:b::3f91:28ac:e654:102b (same!)
```

## RFC 7217 on Linux (systemd-networkd)

Linux kernel 4.6+ and systemd-networkd support RFC 7217 stable privacy addresses.

```bash
# Check if kernel supports RFC 7217
# The feature is in kernel 4.6+
uname -r

# With systemd-networkd: configure in .network file
# /etc/systemd/network/10-eth0.network
cat /etc/systemd/network/10-eth0.network
# [Match]
# Name=eth0
#
# [Network]
# DHCP=no
# IPv6AcceptRA=yes
# IPv6PrivacyExtensions=kernel   ← use kernel default

# For RFC 7217 stable privacy:
# sysctl: net.ipv6.conf.eth0.addr_gen_mode

# Check current addr_gen_mode
cat /proc/sys/net/ipv6/conf/eth0/addr_gen_mode
# 0 = EUI-64
# 1 = random (stable random, pseudo RFC 7217)
# 2 = random via privacy extensions
# 3 = stable-secret (RFC 7217 when stable_secret is set)

# Set to stable-secret mode (RFC 7217)
sudo sysctl -w net.ipv6.conf.eth0.addr_gen_mode=3

# Configure the stable secret (random key)
# This key is used in the hash function
sudo sysctl -w net.ipv6.conf.eth0.stable_secret=\
"fd00::e3a7:b234:9f12:8c56"
# The secret should be a random IPv6 address (used as key material)

# Generate a random secret
python3 -c "import secrets; print('fd00::' + ':'.join(
    f'{secrets.randbelow(65536):04x}' for _ in range(4)))"
```

## RFC 7217 on Linux with iproute2

```bash
# View current stable secret
cat /proc/sys/net/ipv6/conf/eth0/stable_secret
# fd00::e3a7:b234:9f12:8c56  (if configured)
# cat: can't open: Permission denied  (if not set)

# Set stable secret via sysctl
sudo sysctl -w "net.ipv6.conf.eth0.stable_secret=\
fd12:3456:789a:bcde::f012:3456:789a:bcde"

# Set addr_gen_mode to 3 (use stable_secret)
sudo sysctl -w net.ipv6.conf.eth0.addr_gen_mode=3

# Persist configuration
cat >> /etc/sysctl.d/60-ipv6-stable-privacy.conf << 'EOF'
net.ipv6.conf.eth0.addr_gen_mode = 3
net.ipv6.conf.eth0.stable_secret = fd00::e3a7:b234:9f12:8c56
EOF

# Verify new stable privacy address was generated
# (bring interface down and up to regenerate)
sudo ip link set eth0 down
sudo ip link set eth0 up
ip -6 addr show eth0 | grep "scope global"
# inet6 2001:db8::9c23:5f1a:b2e4:7d30/64 scope global dynamic
#    valid_lft ...
# Note: no "ff:fe" in the address = not EUI-64
# Note: stable across reboots on same network
```

## RFC 7217 vs Privacy Extensions

```
Choosing Between RFC 7217 and RFC 8981:

RFC 7217 Stable Privacy:
  Use when:
  - Stability matters (DNS records, firewall rules with host addresses)
  - Rebooting should give the same address
  - Privacy from cross-network tracking is required
  - Applications maintain sessions across reconnects

  Best for: Laptops, workstations, servers that roam between networks

RFC 8981 Privacy Extensions:
  Use when:
  - Maximum privacy is required
  - Address stability is not important
  - Short-lived connections only
  - Preventing address reuse over time is critical

  Best for: Mobile devices, anonymous browsing, temporary connections

Combined deployment (some systems use both):
  - RFC 7217 as the "public" stable address
  - RFC 8981 temporary addresses for outbound connections
  - Incoming connections (servers) use stable address
  - Outbound connections (browsing) use temporary address
```

## Conclusion

RFC 7217 stable privacy addresses provide a middle ground between EUI-64 (stable but trackable) and privacy extensions (private but unstable). The interface identifier is generated by hashing the network prefix, interface, and a per-host secret key, producing a stable pseudo-random address that is the same each time you connect to the same network but different on each network. This provides both session stability and privacy across networks. On Linux, configure with `addr_gen_mode=3` and a `stable_secret`. Modern operating systems use RFC 7217 or similar algorithms as the default SLAAC address generation method.
