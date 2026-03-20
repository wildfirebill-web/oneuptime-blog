# How to Configure IPsec IPv6 on pfSense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, pfSense, VPN, Firewall

Description: Learn how to configure IPv6 IPsec site-to-site VPN tunnels on pfSense, including Phase 1 and Phase 2 settings, firewall rules, and troubleshooting.

## Overview

pfSense uses strongSwan under the hood for IPsec. Configuring IPv6 IPsec in pfSense is done through the web UI under VPN → IPsec. The process involves creating a Phase 1 (IKE SA) and Phase 2 (IPsec SA) entry for each site-to-site tunnel.

## Prerequisites

- pfSense with a global IPv6 address on WAN interface
- Remote peer's IPv6 address
- Matching pre-shared key on both pfSense instances

## Configuring IPv6 IPsec (Web UI)

### Phase 1 (IKE SA Configuration)

Navigate to **VPN → IPsec → Tunnels → Add P1**:

```text
General Information:
  Key Exchange Version: IKEv2
  Internet Protocol:    IPv6
  Interface:            WAN (or your IPv6 uplink)
  Remote Gateway:       2001:db8:gw2::1

Phase 1 Proposal (Authentication):
  Authentication Method: Mutual PSK
  My Identifier:         My IP Address
  Peer Identifier:       Peer IP Address
  Pre-Shared Key:        [your strong key]

Phase 1 Proposal (Encryption):
  Encryption Algorithm: AES 256 bits
  Hash Algorithm:       SHA256
  DH Group:             14 (2048 bit)
  Lifetime:             28800

Advanced Options:
  Dead Peer Detection:  Enable
  DPD Delay:            10
  DPD Max Failures:     3
```

### Phase 2 (IPsec SA / Tunnel)

Click **Show Phase 2 Entries** → **Add P2**:

```text
General Information:
  Mode: Tunnel IPv6

Networks:
  Local Network:  Network - 2001:db8:site1::/48
  Remote Network: Network - 2001:db8:site2::/48

Phase 2 Proposal (SA/Key Exchange):
  Encryption Algorithms: AES256-GCM 128-bit
  PFS Group: 14 (2048 bit)
  Lifetime: 3600
```

### Apply and Connect

Click **Save** then **Apply Changes**.

Navigate to **VPN → IPsec → Status** and click **Connect P1 and P2s** to initiate.

## Firewall Rules for IPsec

### WAN Rules (allow IKE and ESP)

Navigate to **Firewall → Rules → WAN**:

```text
Add rule:
  Action:     Pass
  Interface:  WAN
  Protocol:   UDP
  Source:     2001:db8:gw2::1
  Destination: WAN Address (IPv6)
  Dest Port:  500 (IKE)

Add rule:
  Action:     Pass
  Interface:  WAN
  Protocol:   UDP
  Source:     2001:db8:gw2::1
  Destination: WAN Address (IPv6)
  Dest Port:  4500 (NAT-T)

Add rule:
  Action:     Pass
  Interface:  WAN
  Protocol:   ESP
  Source:     2001:db8:gw2::1
  Destination: WAN Address (IPv6)
```

### IPsec Interface Rules (allow tunnel traffic)

Navigate to **Firewall → Rules → IPsec**:

```text
Add rule:
  Action:     Pass
  Interface:  IPsec
  Protocol:   Any
  Source:     2001:db8:site2::/48
  Destination: 2001:db8:site1::/48
```

## CLI Access (SSH)

pfSense runs FreeBSD. You can verify IPsec from the command line:

```bash
# SSH to pfSense

ssh admin@pfsense.local

# Show IPsec status
ipsec statusall

# Show active connections
ipsec status

# Show security associations
setkey -D   # IPsec SA database

# Show security policy database
setkey -DP  # IPsec SP database

# Check strongSwan logs
clog /var/log/ipsec.log | tail -50

# Monitor IKEv2 negotiation
tail -f /var/log/ipsec.log | grep -E 'IKE|AUTH|CHILD'
```

## IPv6 Routing Through the Tunnel

After the tunnel is up, pfSense should automatically route traffic:

```bash
# On pfSense CLI: Verify route exists
netstat -rn -f inet6 | grep site2
# Should show: 2001:db8:site2::/48  via <tunnel>

# Test connectivity
ping6 2001:db8:site2::1
```

If routes are missing, add static routes under **System → Routing → Static Routes**:

```text
Destination Network: 2001:db8:site2::/48 / 48
Gateway: Create a new IPv6 gateway via Tunnel interface
```

## Troubleshooting

```bash
# From pfSense CLI:
# Restart IPsec
service ipsec restart
# or through web UI: VPN → IPsec → Status → Restart IPsec

# View Phase 1 negotiation details
ipsec statusall | grep -A 10 "site1-to-site2"

# Enable verbose logging
# In pfSense web UI: VPN → IPsec → Advanced Settings
# Increase IKE log level to "diag"

# Check for firewall blocking IPsec
tcpdump -i em0 'udp port 500 or udp port 4500 or proto 50' -n
```

## Summary

pfSense IPv6 IPsec configuration uses Phase 1 (IKEv2, AES-256, SHA-256, DH group 14) and Phase 2 (Tunnel IPv6, AES-GCM-256) settings in the web UI. Ensure firewall rules allow UDP 500, UDP 4500, and ESP (protocol 50) from the remote gateway's IPv6 address to your WAN IPv6 address. Use the IPsec firewall rule interface to allow tunnel traffic. Monitor from **VPN → IPsec → Status** and from the CLI with `ipsec statusall`. pfSense routes through the tunnel automatically based on Phase 2 network definitions.
