# How to Use the Dapr Cryptography API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cryptography, Encryption, Security, API, Key Management

Description: Learn how to use the Dapr Cryptography API to encrypt and decrypt data without managing cryptographic keys directly in your application code.

---

## What Is the Dapr Cryptography API?

The Dapr Cryptography building block provides a standardized API for encrypting and decrypting data without embedding key management logic in your application. Keys are stored in a pluggable key provider (Azure Key Vault, Kubernetes secrets, local storage), and your app calls simple Dapr APIs to perform cryptographic operations.

This separation means you can rotate keys, change providers, and enforce access control at the infrastructure level without changing application code.

## Supported Providers

- Azure Key Vault (recommended for production)
- Kubernetes secrets
- Local key storage (for development)
- JSON Web Key Sets (JWKS)

## Setting Up a Local Key Provider

For development, use the local file provider:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-crypto-provider
spec:
  type: crypto.dapr.localstorage
  version: v1
  metadata:
  - name: path
    value: ./keys
```

Generate a key for local use:

```bash
mkdir keys
# Generate a 256-bit AES key in JWK format
dapr crypto generate-key --type aes --size 256 --output keys/mykey.json
```

## Encrypting Data via HTTP API

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/crypto/my-crypto-provider/encrypt \
  -H "Content-Type: application/octet-stream" \
  -H "dapr-key-name: mykey" \
  -H "dapr-key-wrap-algorithm: AES" \
  --data-binary "Hello, secret world!" \
  -o encrypted.bin
```

## Decrypting Data via HTTP API

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/crypto/my-crypto-provider/decrypt \
  -H "Content-Type: application/octet-stream" \
  -H "dapr-key-name: mykey" \
  --data-binary @encrypted.bin
```

## Using the Go SDK

```go
package main

import (
    "context"
    "fmt"
    dapr "github.com/dapr/go-sdk/client"
)

func main() {
    client, _ := dapr.NewClient()
    defer client.Close()

    plaintext := []byte("sensitive customer data")

    // Encrypt
    encryptOpts := dapr.EncryptRequestOptions{
        ComponentName:    "my-crypto-provider",
        KeyName:          "mykey",
        KeyWrapAlgorithm: "AES",
    }
    encrypted, err := client.Encrypt(context.Background(),
        bytes.NewReader(plaintext), encryptOpts)
    if err != nil {
        panic(err)
    }

    encryptedBytes, _ := io.ReadAll(encrypted)
    fmt.Printf("Encrypted %d bytes\n", len(encryptedBytes))

    // Decrypt
    decryptOpts := dapr.DecryptRequestOptions{
        ComponentName: "my-crypto-provider",
        KeyName:       "mykey",
    }
    decrypted, err := client.Decrypt(context.Background(),
        bytes.NewReader(encryptedBytes), decryptOpts)
    if err != nil {
        panic(err)
    }

    result, _ := io.ReadAll(decrypted)
    fmt.Printf("Decrypted: %s\n", string(result))
}
```

## Using the Python SDK

```python
from dapr.clients import DaprClient
import io

with DaprClient() as d:
    plaintext = b"sensitive customer data"

    # Encrypt
    encrypted = d.encrypt(
        data=io.BytesIO(plaintext),
        options={
            "componentName": "my-crypto-provider",
            "keyName": "mykey",
            "keyWrapAlgorithm": "AES"
        }
    )
    encrypted_bytes = encrypted.read()
    print(f"Encrypted: {len(encrypted_bytes)} bytes")

    # Decrypt
    decrypted = d.decrypt(
        data=io.BytesIO(encrypted_bytes),
        options={
            "componentName": "my-crypto-provider",
            "keyName": "mykey"
        }
    )
    result = decrypted.read()
    print(f"Decrypted: {result.decode()}")
```

## Supported Algorithms

| Operation | Algorithms |
|-----------|-----------|
| Key wrap | AES, RSA-OAEP, RSA-OAEP-256 |
| Data encryption | AES-GCM (256-bit) |
| Signing | Ed25519, RS256, PS256 |

## Summary

The Dapr Cryptography API provides a portable, provider-agnostic interface for encryption and decryption. Applications call simple encrypt/decrypt operations while key management, storage, and rotation are handled externally by the configured provider. This makes it easy to start with local keys in development and switch to Azure Key Vault or Kubernetes secrets in production without code changes.
