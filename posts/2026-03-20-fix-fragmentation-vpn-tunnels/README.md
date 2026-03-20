# How to Fix Fragmentation Issues in VPN Tunnels (GRE, IPsec)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MTU, VPN, GRE, IPsec, Fragmentation, Linux, Networking

Description: Fix packet fragmentation and MTU-related performance issues in GRE, IPsec, and WireGuard VPN tunnels by correctly sizing MTU and configuring MSS clamping.

## Introduction

VPN tunnels add overhead to packets — headers for encapsulation, encryption, and authentication. This reduces the effective MTU inside the tunnel. When tunnel MTU is not properly configured, packets get fragmented inside the tunnel, causing performance degradation, increased CPU load, and in some cases, connection hangs. The fix requires calculating the correct MTU for each tunnel type and applying it consistently.

## Understand Tunnel Overhead

```
Protocol Overhead (reduces available packet size):

GRE tunnel:
  IP header:       20 bytes
  GRE header:       4 bytes minimum (8 with key/seq)
  Total overhead:  24-28 bytes
  Max payload in 1500 MTU network: 1476-1472 bytes

IPsec Tunnel Mode (ESP, AES-256-GCM):
  IP header:       20 bytes
  ESP header:       8 bytes
  IV (AES):        16 bytes
  ESP trailer:      2 bytes
  Auth tag (GCM):  16 bytes
  Padding (avg):   ~8 bytes
  Total overhead:  ~70 bytes
  Max payload:     ~1430 bytes

WireGuard:
  IP header:       20 bytes
  UDP header:       8 bytes
  WireGuard header: 32 bytes
  Auth tag:        16 bytes
  Total overhead:  ~80 bytes
  Max payload:     1420 bytes

VXLAN:
  Outer IP:        20 bytes
  Outer UDP:        8 bytes
  VXLAN header:     8 bytes
  Inner Ethernet:  14 bytes
  Total overhead:  50 bytes
  Max payload (for inner IP): 1450 bytes inner MTU
```

## Fix GRE Tunnel MTU

```bash
# Set GRE tunnel interface MTU:
# Interface: gre0 (or tun0, etc.)

# Check current MTU:
ip link show gre0

# Set correct MTU (1500 - 24 = 1476):
ip link set gre0 mtu 1476

# Permanent (in /etc/network/interfaces or nmcli):
nmcli connection modify gre-tunnel ip.mtu 1476

# For ip_gre kernel module:
ip tunnel add gre0 mode gre remote 10.1.0.2 local 10.1.0.1 ttl 64
ip link set gre0 mtu 1476 up

# Verify with ping through tunnel:
ping -M do -s 1448 remote-host-behind-gre  # 1476 - 28 = 1448
```

## Fix IPsec Tunnel MTU

```bash
# IPsec overhead varies by cipher and mode:
# AES-128-GCM: ~54 bytes overhead
# AES-256-CBC with SHA-256: ~70 bytes overhead

# Set MTU on xfrm interface:
ip link set xfrm0 mtu 1420   # Conservative value that works for most configs

# For StrongSwan:
# In /etc/swanctl/swanctl.conf:
# connections {
#   my-vpn {
#     children {
#       my-child {
#         # No direct MTU option; use interface MTU instead
#       }
#     }
#   }
# }

# Reduce MTU on the underlying physical interface (affects all traffic):
# Don't do this! Use MSS clamping instead for IPsec

# Better: use iptables MSS clamping for IPsec traffic:
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -m policy --pol ipsec --dir in -j TCPMSS --set-mss 1350

iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -m policy --pol ipsec --dir out -j TCPMSS --set-mss 1350
```

## Fix WireGuard MTU

```bash
# WireGuard interface MTU should be:
# Underlying MTU (e.g., 1500) - WireGuard overhead (80) = 1420

# In WireGuard config (/etc/wireguard/wg0.conf):
[Interface]
Address = 10.0.0.1/24
MTU = 1420    # Add this line

# Or set dynamically:
ip link set wg0 mtu 1420

# If underlying path MTU is less than 1500 (e.g., PPPoE at 1492):
# 1492 - 80 = 1412 → use MTU 1412

# Calculate the right MTU:
UNDERLYING_MTU=1500
WG_OVERHEAD=80
WG_MTU=$((UNDERLYING_MTU - WG_OVERHEAD))
echo "Set WireGuard MTU to: $WG_MTU"
ip link set wg0 mtu $WG_MTU
```

## Apply MSS Clamping (Comprehensive Fix)

```bash
# MSS clamping is the most reliable fix:
# Forces TCP to use smaller segments automatically

# For any VPN interface (tun0, wg0, gre0):
# Clamp MSS for traffic going INTO the VPN:
iptables -t mangle -A POSTROUTING -o wg0 -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Clamp MSS for traffic coming FROM the VPN:
iptables -t mangle -A POSTROUTING -o eth0 -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Or set explicit MSS value:
iptables -t mangle -A POSTROUTING -o wg0 -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1380

# Make persistent:
iptables-save > /etc/iptables/rules.v4
```

## Verify Fix

```bash
# Test that large packets work through tunnel:
# From host on one side of tunnel to host on other side:

# Should succeed (at path MTU):
ping -M do -s 1400 10.0.0.2  # Remote host behind VPN
ping -M do -s 1350 10.0.0.2  # More conservative

# Check TCP MSS in connections through tunnel:
ss -tin state established | grep mss
# MSS should be <= 1380 (or your configured value)

# Watch for retransmissions (fragmentation causing drops):
nstat | grep TcpRetrans  # Should not be increasing during VPN transfer
```

## Conclusion

VPN tunnel MTU issues are caused by the overhead each tunnel protocol adds on top of the underlying MTU. Calculate the correct tunnel MTU by subtracting protocol overhead from the underlying path MTU. Set the tunnel interface MTU explicitly with `ip link set tunnelX mtu SIZE`. For TCP, MSS clamping (`--clamp-mss-to-pmtu`) is the most reliable fix as it handles PMTUD failures automatically. For UDP applications through VPN, ensure application payload size accounts for the reduced effective MTU.
