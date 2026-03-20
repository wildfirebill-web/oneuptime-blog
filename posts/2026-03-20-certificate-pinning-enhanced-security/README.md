# How to Implement Certificate Pinning for Enhanced Security

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Security, TLS, Certificate Pinning, HTTPS, Mobile, Application Security

Description: Learn how certificate pinning works, when to use it, and how to implement it in applications to prevent man-in-the-middle attacks.

---

Certificate pinning associates a server with its specific TLS certificate or public key, causing the client to reject any certificate that doesn't match - even if it's signed by a trusted CA. This protects against compromised CAs or MITM attacks.

---

## How Certificate Pinning Works

Standard TLS validation:
1. Server presents certificate
2. Client verifies it's signed by a trusted CA
3. Client proceeds - any valid CA-signed cert is accepted

With certificate pinning:
1. Server presents certificate
2. Client checks if the certificate/public key matches a stored pin
3. Client rejects the connection if it doesn't match - even if CA-valid

---

## Types of Pinning

| Type               | Pins                          | Flexibility |
|--------------------|-------------------------------|-------------|
| Certificate pin    | Full certificate (DER/hash)   | Low         |
| Public key pin     | Subject Public Key Info (SPKI) | Medium     |
| CA pin             | Intermediate or root CA        | High        |

Public key pinning is the most practical - the pin survives certificate renewals as long as the key pair is preserved.

---

## Extract a Public Key Pin

```bash
# Get the SPKI hash for certificate pinning

openssl s_client -connect api.example.com:443 2>/dev/null   | openssl x509 -pubkey -noout   | openssl pkey -pubin -outform DER   | openssl dgst -sha256 -binary   | openssl enc -base64
# Output: abc123xyz...= (your pin)
```

---

## Python Example with Requests

```python
import requests
import ssl
import hashlib
import base64
from cryptography import x509
from cryptography.hazmat.backends import default_backend

EXPECTED_PIN = "abc123xyz...="  # Your computed pin

def check_pin(conn, cert, errnum, depth, ok):
    if depth == 0:
        der_cert = ssl.DER_cert_to_PEM_cert(cert)
        # Compare pin
        pass
    return ok

# Using requests with custom adapter for pinning
# (Use a library like 'certifi' + custom SSL context for production)
```

---

## Android Implementation (OkHttp)

```kotlin
val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            .add("api.example.com", "sha256/abc123xyz...=")
            .add("api.example.com", "sha256/backup-pin...=")  // Backup pin
            .build()
    )
    .build()
```

---

## HTTP Public Key Pinning (HPKP) Header - Deprecated

```text
# HPKP is deprecated, do not use in production
Public-Key-Pins: pin-sha256="abc123..."; max-age=5184000; includeSubDomains
```

HPKP was removed from browsers due to misuse risk. Use application-level pinning instead.

---

## Summary

Certificate pinning prevents MITM attacks by comparing a server's certificate or public key to a stored pin. Always pin the public key (SPKI hash) rather than the full certificate so that certificate renewals don't break the pin. Include at least one backup pin for rotation. Pinning is most appropriate for mobile apps and services where you control both client and server.
