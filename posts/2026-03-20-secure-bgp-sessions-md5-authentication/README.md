# How to Secure BGP Sessions with MD5 Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, MD5, Authentication, Security, Cisco IOS, TCP-AO

Description: Learn how to configure MD5 authentication on BGP sessions to protect against session hijacking and spoofed TCP RST attacks.

## Why BGP Authentication Matters

BGP runs over TCP port 179. Without authentication, an attacker on the network path can inject spoofed TCP RST packets to tear down BGP sessions, or inject forged BGP UPDATE messages. MD5 authentication (RFC 2385) adds a cryptographic signature to each TCP segment, making these attacks infeasible.

## Step 1: Configure MD5 Authentication on Cisco IOS

Both peers must use the same password. The password is case-sensitive:

```
! On Router A (AS 65001)
router bgp 65001
 neighbor 203.0.113.2 remote-as 65002
 ! Add MD5 password - must match Router B exactly
 neighbor 203.0.113.2 password SecureBGP@2026!

! On Router B (AS 65002)
router bgp 65002
 neighbor 203.0.113.1 remote-as 65001
 ! Same password as Router A
 neighbor 203.0.113.1 password SecureBGP@2026!
```

If passwords don't match, the TCP handshake fails silently and the session stays in Active state.

## Step 2: Verify MD5 Is Active

```
Router# show ip bgp neighbors 203.0.113.2

BGP neighbor is 203.0.113.2, remote AS 65002
  ...
  Outgoing update AS path filter: none
  Enhanced refresh request: enabled, limit: 25
  Incoming update AS path filter: none
  MD5 authentication enabled, keystring 08264E4D0A2F
```

The `MD5 authentication enabled` line confirms authentication is negotiated.

## Step 3: Use Encrypted Password Storage

Store passwords in encrypted form in the configuration to prevent cleartext exposure in `show running-config`:

```
! Use type 7 encryption (weak, Cisco reversible) - minimum protection
Router(config)# service password-encryption

! After enabling service password-encryption, the password appears as:
! neighbor 203.0.113.2 password 7 082E4A4D0B2F1B3D

! For stronger storage, use type 6 encryption (AES-256)
Router(config)# key config-key password-encrypt MyMasterKey!
Router(config)# password encryption aes
```

Note: Type 7 encryption is easily reversed. Type 6 is much stronger.

## Step 4: MD5 Authentication on FRRouting (Linux)

```bash
# In FRR vtysh or /etc/frr/bgpd.conf

router bgp 65001
 neighbor 203.0.113.2 remote-as 65002
 neighbor 203.0.113.2 password SecureBGP@2026!
```

## Step 5: Harden with TTL Security (GTSM)

Generalized TTL Security Mechanism (GTSM, RFC 5082) complements MD5 by setting the expected TTL of incoming BGP packets to 255. Spoofed packets from the Internet arrive with a lower TTL and are dropped at the kernel:

```
! Configure TTL security - expect packets with TTL >= 254 (1 hop away)
router bgp 65001
 neighbor 203.0.113.2 ttl-security hops 1
```

For iBGP multihop sessions, adjust the hop count accordingly.

## Step 6: Consider TCP Authentication Option (TCP-AO)

TCP-AO (RFC 5925) is the successor to MD5 authentication. It supports multiple keys and stronger algorithms. Cisco IOS XE supports TCP-AO:

```
! Define a TCP-AO key chain
key chain BGP_AO_KEYS
 key 1
  key-string AO_Secure_Key_2026!
  cryptographic-algorithm hmac-sha-256

! Apply to BGP neighbor
router bgp 65001
 neighbor 203.0.113.2 ao BGP_AO_KEYS
```

TCP-AO is preferred over MD5 for new deployments where supported.

## Troubleshooting Authentication Failures

If the session stays in Active state after adding authentication:

```
! Check for auth failure messages in logs
Router# show log | include MD5|AUTH|BGP

! Common causes:
! - Password mismatch (typos, case sensitivity)
! - One side has authentication, the other doesn't
! - TTL-security hop mismatch
! - ACL blocking TCP 179

! Test basic TCP connectivity (ignore MD5 error if present)
Router# telnet 203.0.113.2 179
```

## Conclusion

MD5 authentication on BGP sessions is a baseline security requirement for all production BGP peerings. Configure matching passwords on both sides, verify with `show ip bgp neighbors`, and combine with GTSM TTL security for defense in depth. Consider migrating to TCP-AO for new deployments where your platform supports it.
