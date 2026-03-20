# How to Understand IKE Phase 1 and Phase 2 Negotiation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPsec, IKE, IKEv2, Phase 1, Phase 2, VPN, Networking

Description: Understand how IKE Phase 1 (IKE SA) and Phase 2 (Child SA) negotiation works to establish IPsec VPN tunnels, including key exchange and algorithm selection.

IKE (Internet Key Exchange) is the protocol that negotiates and establishes IPsec security associations. Understanding its phases helps diagnose VPN failures and tune performance.

## IKEv1 vs. IKEv2

| Feature | IKEv1 | IKEv2 |
|---|---|---|
| Phases | Phase 1 + Phase 2 | IKE SA + Child SA |
| Messages | 6 (main mode) or 3 (aggressive) | 4 (initial exchange) |
| MOBIKE | No | Yes |
| EAP auth | No | Yes |
| DoS resistance | Lower | Higher |
| Recommended? | Legacy | Yes |

## IKEv2 Negotiation Flow

```
Initiator                                    Responder
    |                                             |
    |--- IKE_SA_INIT (propose algorithms) ------->|
    |     SA proposals (AES-256, SHA-256, DH-2048)|
    |     Nonce, DH public value                  |
    |                                             |
    |<-- IKE_SA_INIT (accept algorithms) ---------|
    |     Chosen algorithms                       |
    |     Nonce, DH public value                  |
    |                                             |
    [Both sides compute shared secret via DH]     |
    |                                             |
    |--- IKE_AUTH (prove identity) ------------->|
    |     ID, Certificate or PSK hash            |
    |     Child SA proposal (ESP AES-256)         |
    |     Traffic selectors (subnets)             |
    |                                             |
    |<-- IKE_AUTH (accept) ----------------------|
    |     Authenticated, Child SA installed       |
    |                                             |
    [IPsec tunnel established, data flows]
```

## strongSwan Configuration for Each Phase

```conf
# /etc/ipsec.conf — phase parameters

conn my-vpn
    # Phase 1 (IKE SA) parameters
    ike=aes256-sha256-modp2048!
    #    ^algorithm  ^hash   ^DH group

    # Phase 2 (Child SA / ESP) parameters
    esp=aes256-sha256!
    #   ^algorithm  ^hash (no DH here by default; add -modp2048 for PFS)

    # Lifetime settings
    ikelifetime=28800s    # Phase 1 SA lifetime: 8 hours
    lifetime=3600s        # Phase 2 SA lifetime: 1 hour
    margintime=540s       # Renegotiate 9 minutes before expiry
    keyingtries=3         # Try 3 times before giving up
```

## Phase 1: IKE SA Parameters

Phase 1 establishes an encrypted, authenticated channel for IKE traffic:

```conf
# Format: encryption-integrity-dh_group
# Recommended strong options:
ike=aes256gcm128-prfsha384-ecp384!   # AES-GCM + SHA-384 + ECC P-384
ike=aes256-sha256-modp2048!           # AES-256 + SHA-256 + DH group 14
ike=aes256-sha512-modp4096!           # Maximum security
```

## Phase 2: Child SA (ESP) Parameters

Phase 2 establishes the IPsec SAs that protect data:

```conf
# Format: encryption-integrity
# With Perfect Forward Secrecy (add DH group):
esp=aes256gcm128-ecp384!             # AEAD mode (best)
esp=aes256-sha256-modp2048!          # PFS with DH group 14
esp=aes256-sha256!                    # No PFS (faster but less secure)
```

## Observing Negotiation with Logging

```bash
# Enable detailed IKE logging in strongSwan
# In /etc/strongswan.d/charon.conf:
# filelog {
#   /var/log/charon.log {
#     default = 1
#     ike = 3
#     knl = 1
#   }
# }

sudo ipsec stroke loglevel ike 4

# Watch the negotiation in real time
sudo journalctl -u strongswan -f | grep -E "IKE|SA|proposal"
```

## Common Negotiation Failures

```bash
# No proposal chosen — algorithm mismatch
# Fix: ensure both sides have matching ike= and esp= parameters

# Deadlock in authentication
# Fix: verify PSK/certificate on both sides

# Check what was proposed and accepted
sudo ipsec statusall | grep -A3 "IKE proposal"
```

## Checking Active SAs

```bash
# View IKE SAs (Phase 1)
sudo ipsec statusall | grep "IKE SA"

# View Child SAs (Phase 2)
sudo ipsec statusall | grep "CHILD SA"

# Kernel-level XFRM states
sudo ip xfrm state list

# Manually rekey
sudo ipsec rekey <connection-name>
```

Understanding IKE phases helps immediately identify whether a VPN failure is at the authentication level (Phase 1) or the tunnel encryption level (Phase 2).
