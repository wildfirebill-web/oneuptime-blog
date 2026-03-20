# How to Understand SEND RSA Signature Option

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SEND, RSA Signature, NDP Security, IPv6 Security, RFC 3971, Cryptography

Description: Understand how the SEND RSA Signature option (Type 12) authenticates NDP messages with digital signatures, its format, and how verification works.

## Introduction

The RSA Signature option (ICMPv6 NDP option Type 12, defined in RFC 3971) provides message authentication for SEND-enabled NDP packets. It contains an RSA digital signature over the NDP message and the sender's CGA parameters, allowing any receiver to verify that the message was signed by the holder of the private key corresponding to the CGA. The RSA Signature option is one of four SEND options (CGA, RSA Signature, Timestamp, Nonce) that together protect NDP messages from spoofing and replay attacks.

## RSA Signature Option Format

```
RSA Signature Option (Type 12) Wire Format:

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type=12   |    Length     |       Reserved                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                       Key Hash (128 bits)                     .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
.                    Digital Signature                          .
.                    (variable length)                          .
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Padding (variable)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Fields:
  Type:    8 bits = 12 (RSA Signature)
  Length:  8 bits = total option length in units of 8 bytes
  Reserved: 16 bits = 0
  Key Hash: 128-bit (16-byte) SHA-1 hash of the public key
           Used to identify which public key signed this message
  Digital Signature: RSA signature (variable, depends on key size)
           For 1024-bit RSA: 128 bytes
           For 2048-bit RSA: 256 bytes
  Padding: Zero padding to align to 8-byte boundary
```

## What Is Signed

The RSA signature covers specific fields of the NDP message to prevent tampering.

```
Signed Data for RSA Signature Computation (RFC 3971 Section 5.2):

signed_data =
  source_IPv6_address (16 bytes)       ← from IPv6 header
  + destination_IPv6_address (16 bytes) ← from IPv6 header
  + Message Type (8 bits = ICMPv6 type) ← from ICMPv6 header
  + Reserved (24 bits = 0)
  + ICMPv6 message body
    (excluding the RSA Signature option itself)
  + CGA Parameters Data Structure
    (the CGA Parameters, not the CGA option)
  + Nonce option data (if present)
  + Timestamp option data (if present)

The RSA Signature option itself is excluded from the signed data.
The Key Hash in the RSA Signature option identifies which public
key to use for verification.

Signature algorithm: RSA with SHA-1 (RSASSA-PKCS1-v1_5-SHA1)
```

## RSA Signature Verification Process

```
SEND RSA Signature Verification Steps (RFC 3971 Section 5.3):

Step 1: Receive NDP message with SEND options
  - Extract CGA option (contains public key)
  - Extract RSA Signature option
  - Extract Timestamp option
  - Extract Nonce option (if applicable)

Step 2: Verify Timestamp (replay attack prevention)
  - Current time - Timestamp < allowed delta (default: 300 sec)
  - If too old or too far in future: REJECT

Step 3: Verify Nonce (reflection attack prevention)
  - For solicited NA: Nonce must match the Nonce in the NS
  - If nonce mismatch: REJECT

Step 4: Verify Key Hash
  - Compute SHA-1 hash of public key from CGA option
  - Compare with Key Hash in RSA Signature option
  - If mismatch: REJECT (key doesn't match claimed hash)

Step 5: Verify CGA
  - Use CGA verification algorithm (RFC 3972)
  - Verify that source IPv6 address is derived from public key
  - If CGA invalid: REJECT

Step 6: Verify RSA Signature
  - Reconstruct signed_data (as specified above)
  - Verify RSA signature using public key from CGA option
  - RSA-verify(public_key, signature, SHA-1(signed_data))
  - If signature invalid: REJECT

Step 7: Accept message
  - All checks passed: message is authenticated
  - Process NDP message normally
```

## RSA Signature Verification in Python

This example shows the verification logic at a conceptual level.

```python
import hashlib
import struct
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.backends import default_backend

def verify_send_rsa_signature(
    src_addr: bytes,      # 16-byte IPv6 source address
    dst_addr: bytes,      # 16-byte IPv6 destination address
    icmpv6_type: int,     # ICMPv6 message type (133-137)
    icmpv6_body: bytes,   # ICMPv6 body without RSA signature option
    cga_params: bytes,    # CGA Parameters Data Structure
    timestamp_data: bytes, # Timestamp option data
    nonce_data: bytes,    # Nonce option data (may be empty)
    public_key_der: bytes, # DER-encoded RSA public key from CGA option
    signature: bytes,      # RSA signature from signature option
) -> bool:
    """
    Verify SEND RSA Signature per RFC 3971 Section 5.2.
    Returns True if signature is valid.
    """
    # Construct signed data
    # Type byte + 3 reserved bytes
    type_reserved = bytes([icmpv6_type, 0, 0, 0])

    signed_data = (
        src_addr +
        dst_addr +
        type_reserved +
        icmpv6_body +
        cga_params +
        timestamp_data +
        nonce_data
    )

    # Load public key
    public_key = serialization.load_der_public_key(
        public_key_der, backend=default_backend()
    )

    # Verify RSA signature (RSASSA-PKCS1-v1_5 with SHA-1)
    try:
        public_key.verify(
            signature,
            signed_data,
            padding.PKCS1v15(),
            hashes.SHA1()
        )
        return True
    except Exception:
        return False
```

## Timestamp Option (Type 13) Format

The Timestamp option works with the RSA Signature to prevent replay attacks.

```
Timestamp Option (Type 13):

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type=13   |    Length=2   |         Reserved              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Timestamp (64-bit NTP timestamp)                |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Timestamp: 64-bit NTP timestamp (seconds since Jan 1, 1900)
  Top 32 bits: seconds
  Bottom 32 bits: fractional seconds

Verification:
  Receiver accepts if: |current_time - timestamp| < delta_time
  Default delta_time: 300 seconds (configurable)
  Requires: time synchronization between sender and receiver (NTP)
```

## Nonce Option (Type 14) Format

```
Nonce Option (Type 14):

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type=14   |    Length     |       Nonce Value ...         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    ... Nonce Value (variable)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Nonce Value: random value chosen by NS sender
  Minimum 6 bytes, typically 8 bytes
  Must be echoed in corresponding NA (Solicited NA)
  Prevents reflection: old captured NAs cannot be replayed

Usage:
  NS includes Nonce option with random value
  Corresponding NA must include Nonce option with same value
  Verifier checks: NA.Nonce == NS.Nonce → valid response
```

## Conclusion

The SEND RSA Signature option provides cryptographic integrity and origin authentication for NDP messages. It signs the NDP message content along with the source address, CGA parameters, timestamp, and nonce to prevent tampering, replay, and reflection attacks. Verification requires: timestamp freshness check, nonce validation for solicited responses, CGA verification (address is derived from the public key), and RSA signature verification. While computationally expensive due to RSA operations per NDP message, the RSA Signature option provides the strongest available NDP authentication when SEND is deployed.
