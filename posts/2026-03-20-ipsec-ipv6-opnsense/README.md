# How to Configure IPsec IPv6 on OPNsense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, OPNsense, VPN, Firewall

Description: Step-by-step guide to configuring IPv6 IPsec site-to-site VPN on OPNsense, including Phase 1 and Phase 2 configuration, firewall rules, and monitoring.

## Overview

OPNsense uses strongSwan for IPsec and provides a web UI for configuration under VPN → IPsec. It supports IKEv2 with IPv6 endpoints for site-to-site tunnels. OPNsense configuration is similar to pfSense but with a different UI layout and some naming differences.

## Configuring IPv6 IPsec on OPNsense

### Phase 1 Configuration

Navigate to **VPN → IPsec → Connections → Add**:

```text
General Settings:
  Connection: Enabled
  Description: IPv6-Site-To-Site
  Key Exchange: IKEv2
  Internet Protocol: IPv6

Remote Gateway:
  Remote gateway: 2001:db8:gw2::1

Authentication:
  Authentication method: Pre-Shared Key
  My identifier:         My IP address
  Peer identifier:       Peer IP address
  Pre-shared key:        [strong PSK]

Phase 1 Proposal:
  Encryption algorithms: AES-256
  Hash algorithms:       SHA256
  DH Groups:             14 (2048 bit MODP)
  Lifetime:              28800

Dead Peer Detection:
  Enable:  Checked
  Delay:   30
  Maxfail: 3
```

### Phase 2 Configuration

Under the Phase 1 entry, click **Add Phase 2**:

```text
General:
  Description: site1-to-site2
  Mode: Tunnel IPv6

Local Network:
  Type: Network
  Address: 2001:db8:site1::/48

Remote Network:
  Type: Network
  Address: 2001:db8:site2::/48

Phase 2 Proposal:
  Encryption Algorithms: AES-GCM-256 (128-bit tag)
  PFS key group: 14 (2048 bit MODP)
  Lifetime: 3600
```

Click **Save** and then **Apply Changes**.

## Firewall Rules

### WAN Interface Rules

Navigate to **Firewall → Rules → WAN**:

```text
# Rule 1: Allow IKEv2

Action:           Pass
Interface:        WAN
Direction:        in
TCP/IP Version:   IPv6
Protocol:         UDP
Source:           2001:db8:gw2::1/128
Destination:      WAN address
Destination port: 500 (ISAKMP)
Description:      Allow IKEv2 from remote gateway

# Rule 2: Allow NAT-T
(Same as above but port 4500)

# Rule 3: Allow ESP
Action:           Pass
Protocol:         ESP (50)
Source:           2001:db8:gw2::1/128
```

### IPsec Interface Rules

Navigate to **Firewall → Rules → IPsec**:

```text
Action:           Pass
Interface:        IPsec
TCP/IP Version:   IPv6
Protocol:         Any
Source:           2001:db8:site2::/48
Destination:      2001:db8:site1::/48
Description:      Allow tunnel traffic from Site2

```

## Monitoring from OPNsense UI

Navigate to **VPN → IPsec → Status Overview**:

```text
Active Tunnels:
  Connection        Remote Gateway         State
  IPv6-S2S          2001:db8:gw2::1        ESTABLISHED
  Child SAs:
    site1-to-site2  2001:db8:site1::/48 ↔ 2001:db8:site2::/48
    Bytes in: 45892    Bytes out: 38422
```

Click **Connect** to initiate the tunnel or **Disconnect** to terminate.

## CLI Verification (SSH)

OPNsense runs FreeBSD with strongSwan:

```bash
# SSH to OPNsense
ssh root@opnsense.local

# Show strongSwan status
ipsec statusall

# List active SAs
swanctl --list-sas

# List active connections
swanctl --list-conns

# Initiate manually
swanctl --initiate conn:IPv6-Site-To-Site

# Ping through tunnel
ping6 -c 3 2001:db8:site2::1

# View IPsec logs
grep -i ipsec /var/log/system.log | tail -50

# tcpdump: Verify ESP on WAN
tcpdump -i vtnet0 'ip6 proto 50' -n -c 10
```

## Key Differences: OPNsense vs pfSense for IPv6 IPsec

| Feature | OPNsense | pfSense |
|---------|----------|---------|
| UI Location | VPN → IPsec → Connections | VPN → IPsec → Tunnels |
| IKEv2 support | Yes | Yes |
| IPv6 support | Yes | Yes |
| Logging | System log + swanctl | /var/log/ipsec.log |
| Plugin management | OPNsense plugins | pfSense packages |

## Routes After Tunnel Establishment

```bash
# Verify route was added
netstat -rn -f inet6 | grep 'site2'
# or
route -n get -inet6 2001:db8:site2::1

# If route is missing, add static route:
# System → Routes → Configuration → Add
# Destination: 2001:db8:site2::/48
# Gateway: dynamic-VPN-gateway or tunnel interface
```

## Summary

OPNsense IPv6 IPsec configuration follows the same Phase 1/Phase 2 structure as pfSense. Set Internet Protocol to IPv6 in Phase 1, use AES-256-GCM in Phase 2, and ensure firewall rules allow UDP 500, UDP 4500, and ESP from the remote gateway. Monitor from **VPN → IPsec → Status Overview** or via CLI with `swanctl --list-sas`. Routes to remote sites are automatically installed when Phase 2 is established. Use `swanctl --initiate` from CLI if the tunnel doesn't auto-start.
