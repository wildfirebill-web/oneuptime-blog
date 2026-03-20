# How to Understand the Encapsulating Security Payload (ESP) in IPv6 (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, ESP, Security, Encryption

Description: Understand the Encapsulating Security Payload (ESP) extension header in IPv6, how it provides confidentiality and authentication for IPv6 traffic, and how to configure it with Linux IPsec.

## Introduction

The Encapsulating Security Payload (ESP, Next Header = 50) is the workhorse of IPsec security for IPv6. It provides confidentiality (encryption), data origin authentication, connectionless integrity, anti-replay protection, and limited traffic flow confidentiality. ESP is more widely used than AH because it provides both encryption and authentication in a single header, and it is compatible with NAT traversal (unlike AH).

## ESP Packet Structure

```text
ESP wraps the payload with header and trailer:

[IPv6 Header][ESP Header][Encrypted Payload][ESP Trailer][ESP Auth Data]

ESP Header (8 bytes):
  - Security Parameters Index (SPI): 4 bytes
  - Sequence Number: 4 bytes

ESP Encrypted Payload:
  - Original upper-layer data (TCP/UDP/etc.)

ESP Trailer:
  - Padding (0-255 bytes to align to cipher block boundary)
  - Pad Length: 1 byte
  - Next Header: 1 byte (protocol of encrypted payload, e.g., 6=TCP)

ESP Auth Data (variable):
  - Integrity Check Value over SPI + Sequence + Encrypted Payload + Trailer
  - Typically 12-16 bytes (truncated HMAC)
```

## Transport vs Tunnel Mode

```text
Transport Mode (end-to-end between two hosts):
  [IPv6][ESP Header][Encrypted: TCP + Data][ESP Trailer][ICV]
  The IPv6 header is visible; only payload is encrypted

Tunnel Mode (VPN between two gateways):
  [New IPv6][ESP Header][Encrypted: Old IPv6 + TCP + Data][ESP Trailer][ICV]
  The entire original packet (including IP header) is encrypted
  Used for site-to-site VPNs
```

## Configuring ESP with Linux Kernel IPsec

```bash
# Configure ESP between two hosts using ip xfrm

# On Host A (2001:db8::1):

# Add Security Association (outbound to Host B)
sudo ip xfrm state add \
    src 2001:db8::1 dst 2001:db8::2 \
    proto esp spi 0xABCD1234 \
    mode transport \
    enc "aes" 0x000102030405060708090a0b0c0d0e0f \
    auth-trunc "hmac(sha256)" \
    0x0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20 \
    128

# Add outbound policy
sudo ip xfrm policy add \
    src 2001:db8::1 dst 2001:db8::2 \
    dir out \
    tmpl src 2001:db8::1 dst 2001:db8::2 \
    proto esp mode transport

# Verify ESP is applied
ping6 -c 3 2001:db8::2
sudo tcpdump -i eth0 "ip6[6] == 50"  # ESP packets
```

## strongSwan Configuration (Modern IPsec)

```text
# /etc/ipsec.conf - strongSwan site-to-site VPN over IPv6

config setup
    charondebug="ike 1, net 1"

conn ipv6-tunnel
    type=tunnel
    keyexchange=ikev2
    left=2001:db8::1
    leftsubnet=2001:db8:site1::/48
    right=2001:db8::2
    rightsubnet=2001:db8:site2::/48
    ike=aes256-sha256-modp2048!
    esp=aes256-sha256!
    auto=start
```

## ESP Cipher Suites Commonly Used

```bash
# Modern ESP cipher suites (RFC 8221 recommended):
# Combined mode (encryption + authentication in one):
#   aes128gcm16   - AES-GCM 128-bit (recommended)
#   aes256gcm16   - AES-GCM 256-bit

# Separate encryption + auth:
#   aes128-sha256  - AES-128-CBC + HMAC-SHA-256
#   aes256-sha256  - AES-256-CBC + HMAC-SHA-256

# Check supported algorithms on Linux
sudo ip xfrm state list
cat /proc/crypto | grep -A 2 "name.*gcm"
```

## NAT Traversal (NAT-T) with ESP

Unlike AH, ESP is compatible with NAT using NAT-T (UDP port 4500 encapsulation):

```bash
# NAT-T wraps ESP in UDP for NAT compatibility
# IKE: UDP 500 (initial) → UDP 4500 (after NAT detected)
# ESP after NAT-T: [UDP 4500][ESP][Encrypted Payload]

# Verify NAT-T is being used
sudo tcpdump -i eth0 "udp port 4500"
```

## ESP vs AH Summary

| Feature | ESP | AH |
|---|---|---|
| Encryption | Yes | No |
| Integrity | Yes (payload only) | Yes (entire packet inc. IP header) |
| NAT compatible | Yes (with NAT-T) | No |
| Header size | 8 bytes + auth | 12 bytes + auth |
| Common use | Yes (preferred) | Rare |
| Tunnel mode | Yes | Yes |

## Conclusion

ESP is the standard choice for IPv6 IPsec deployments, providing encryption, authentication, and anti-replay protection in transport or tunnel mode. Combined mode ciphers like AES-GCM provide both encryption and authentication efficiently in a single pass. ESP's compatibility with NAT traversal makes it practical in mixed environments, unlike AH which breaks when source or destination addresses are translated. For most VPN and secure communication needs, ESP with AES-256-GCM is the recommended configuration.
