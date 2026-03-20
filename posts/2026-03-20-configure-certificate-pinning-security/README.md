# How to Configure Certificate Pinning for Enhanced Security

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Certificate Pinning, TLS, HPKP, Security, HTTPS, Mobile Security, API Security

Description: Learn how to implement certificate pinning in mobile apps, HTTP clients, and APIs to prevent man-in-the-middle attacks by binding connections to specific certificates or public keys.

---

Certificate pinning binds a client to a specific certificate or public key, rejecting connections even if a valid CA-signed certificate is presented by an impostor.

## How Certificate Pinning Works

```
Without pinning:
  Client → trusts any certificate signed by a known CA → MITM possible with rogue CA

With pinning:
  Client → compares server cert/public key to stored pin → rejects if no match
```

## Extracting a Public Key Pin

```bash
# Get the server certificate
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform der \
  | openssl dgst -sha256 -binary \
  | openssl enc -base64

# Output (example):
# abc123def456ghi789jkl012mno345pqr678stu901vwx234yz==
```

## Certificate Pinning in curl

```bash
# Pin using the server's certificate file
curl --pinnedpubkey "sha256//abc123def456ghi789jkl012mno345pqr678stu901vwx234yz==" \
     https://api.example.com

# Pin using a local certificate file
curl --cacert /etc/ssl/certs/api-pinned.pem https://api.example.com
```

## Certificate Pinning in Python

```python
import requests

# Using requests with certificate verification
response = requests.get(
    "https://api.example.com",
    verify="/etc/ssl/certs/api-ca.pem"  # Pin to specific CA cert
)

# For strict public key pinning, use requests-toolbelt or httpx
# with a custom SSL adapter that verifies the public key hash
```

## Certificate Pinning in Go

```go
package main

import (
    "crypto/sha256"
    "crypto/tls"
    "crypto/x509"
    "encoding/base64"
    "fmt"
    "net/http"
)

var pinnedKey = "abc123def456ghi789jkl012mno345pqr678stu901vwx234yz=="

func main() {
    tlsConfig := &tls.Config{
        VerifyConnection: func(cs tls.ConnectionState) error {
            for _, cert := range cs.PeerCertificates {
                pubKeyDer, _ := x509.MarshalPKIXPublicKey(cert.PublicKey)
                hash := sha256.Sum256(pubKeyDer)
                pin := base64.StdEncoding.EncodeToString(hash[:])
                if pin == pinnedKey {
                    return nil
                }
            }
            return fmt.Errorf("certificate pin mismatch")
        },
    }

    client := &http.Client{
        Transport: &http.Transport{TLSClientConfig: tlsConfig},
    }
    resp, err := client.Get("https://api.example.com")
    _ = resp
    _ = err
}
```

## Certificate Pinning in Android (OkHttp)

```kotlin
val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            .add("api.example.com", "sha256/abc123def456ghi789jkl012mno345pqr678stu901vwx234yz==")
            .add("api.example.com", "sha256/backupPin456...")  // Always include a backup pin
            .build()
    )
    .build()
```

## Best Practices

| Practice | Why |
|----------|-----|
| Pin the public key, not the certificate | Survives cert renewal with same key pair |
| Always include a backup pin | Allows key rotation without app update |
| Set an expiry date for pins | Prevents lockout if pin becomes outdated |
| Monitor for pin failures | Detect MITM attempts in production |

## Key Takeaways

- Pin the public key (SPKI hash) rather than the full certificate to survive renewals.
- Always configure a backup pin to allow key rotation without breaking clients.
- Use certificate pinning for high-value API endpoints (banking, authentication, payment).
- Test pinning in staging before production — a misconfigured pin breaks all client connectivity.
