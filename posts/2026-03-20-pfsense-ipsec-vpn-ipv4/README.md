# How to Set Up IPsec VPN for IPv4 on pfSense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: pfSense, IPsec, VPN, IPv4, Site-to-Site, IKEv2, Security

Description: Configure a site-to-site IPsec VPN on pfSense using IKEv2 with pre-shared keys to connect two IPv4 networks, including Phase 1, Phase 2, and firewall rules.

## Introduction

pfSense uses strongSwan under the hood for IPsec. Site-to-site configuration requires matching Phase 1 (IKE) and Phase 2 (IPsec) parameters on both endpoints.

## Phase 1 Configuration

Navigate to **VPN > IPsec > Tunnels > Add P1**:

```
Key Exchange version:   IKEv2
Internet Protocol:      IPv4
Interface:              WAN
Remote Gateway:         203.0.113.2    (peer WAN IP)
Authentication Method:  Mutual PSK
Pre-Shared Key:         StrongIPsecKey123

Phase 1 Proposal:
  Encryption Algorithm: AES 256
  Hash Algorithm:       SHA256
  DH Group:             14 (2048 bit)
  Lifetime:             28800
```

## Phase 2 Configuration

Click **Show Phase 2 Entries > Add P2**:

```
Mode:            Tunnel IPv4
Local Network:   192.168.1.0/24   (Site A LAN)
Remote Network:  192.168.2.0/24   (Site B LAN)

Phase 2 Proposal:
  Encryption Algorithms: AES 256
  Hash Algorithms:       SHA256
  PFS Key Group:         14
  Lifetime:              3600
```

## Firewall Rules

Navigate to **Firewall > Rules > WAN > Add**:
```
Protocol: ESP
Source:   203.0.113.2
Description: Allow IPsec ESP from peer
```
```
Protocol: UDP
Source:   203.0.113.2
Port:     500, 4500
Description: Allow IKE from peer
```

Navigate to **Firewall > Rules > IPsec > Add**:
```
Action: Pass
Source: 192.168.2.0/24
Destination: 192.168.1.0/24
Description: Allow traffic from Site B
```

## Site B Configuration (Mirror of Site A)

Repeat the same Phase 1 and Phase 2 configuration on Site B with:
- Remote gateway: `203.0.113.1` (Site A WAN IP)
- Local network: `192.168.2.0/24`
- Remote network: `192.168.1.0/24`
- Same PSK and proposal parameters

## NAT Exclusion

Navigate to **Firewall > NAT > Outbound**:
- Switch to **Hybrid** mode
- Add rule:
  ```
  Interface: WAN
  Source: 192.168.1.0/24
  Destination: 192.168.2.0/24
  Translation: No NAT / No BINAT
  ```

## Verify IPsec

Navigate to **Status > IPsec**:
- Phase 1: Connected (green)
- Phase 2: Connected (green)

```bash
# pfSense CLI
ipsec statusall
ping -S 192.168.1.1 192.168.2.1
```

## Conclusion

pfSense IPsec site-to-site VPN requires matching IKEv2 Phase 1 proposals, Phase 2 with local/remote network definitions, firewall rules for ESP/IKE traffic, and a NAT exclusion to prevent the VPN traffic from being masqueraded. Both sites must have identical cryptographic parameters and mirrored network definitions.
