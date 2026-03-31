# How to Use the Dapr Cryptography API Reference

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cryptography, API, Encryption, Security

Description: A practical reference for the Dapr Cryptography API covering encrypt, decrypt, sign, and verify operations using managed key vaults.

---

## Overview

The Dapr Cryptography API lets applications encrypt, decrypt, sign, and verify data using keys managed in a key vault backend. Applications never handle raw key material - the Dapr sidecar delegates all cryptographic operations to the vault, keeping keys out of application memory.

## Supported Backends

- Azure Key Vault
- HashiCorp Vault (with Transit secrets engine)
- Local file-based keys (development only)

## Component Definition

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: myvault
spec:
  type: crypto.azure.keyvault
  version: v1
  metadata:
    - name: vaultUri
      value: https://my-vault.vault.azure.net/
    - name: azureClientId
      value: "your-client-id"
    - name: azureTenantId
      value: "your-tenant-id"
    - name: azureClientSecret
      secretKeyRef:
        name: azure-secret
        key: clientSecret
```

## Encrypting Data (Go SDK)

```go
package main

import (
    dapr "github.com/dapr/go-sdk/client"
    "context"
    "bytes"
)

func encryptData(plaintext []byte) ([]byte, error) {
    client, _ := dapr.NewClient()
    defer client.Close()

    out, err := client.Encrypt(context.Background(),
        bytes.NewReader(plaintext),
        dapr.EncryptOptions{
            ComponentName:    "myvault",
            KeyName:          "my-encryption-key",
            Algorithm:        "RSA-OAEP-256",
        },
    )
    if err != nil {
        return nil, err
    }

    var buf bytes.Buffer
    buf.ReadFrom(out)
    return buf.Bytes(), nil
}
```

## Decrypting Data (Go SDK)

```go
func decryptData(ciphertext []byte) ([]byte, error) {
    client, _ := dapr.NewClient()
    defer client.Close()

    out, err := client.Decrypt(context.Background(),
        bytes.NewReader(ciphertext),
        dapr.DecryptOptions{
            ComponentName: "myvault",
            KeyName:       "my-encryption-key",
        },
    )
    if err != nil {
        return nil, err
    }

    var buf bytes.Buffer
    buf.ReadFrom(out)
    return buf.Bytes(), nil
}
```

## Signing Data (Python SDK)

```python
from dapr.clients import DaprClient

with DaprClient() as client:
    sign_response = client.sign(
        data=b"important document content",
        component_name="myvault",
        key_name="my-signing-key",
        algorithm="PS256"
    )
    signature = sign_response.signature
```

## Verifying a Signature

```python
with DaprClient() as client:
    verify_response = client.verify(
        data=b"important document content",
        signature=signature,
        component_name="myvault",
        key_name="my-signing-key",
        algorithm="PS256"
    )
    if verify_response.success:
        print("Signature is valid")
    else:
        print("Signature verification failed")
```

## Supported Algorithms

| Use Case | Algorithms |
|---|---|
| Asymmetric encryption | RSA-OAEP, RSA-OAEP-256 |
| Symmetric encryption | AES-CBC, AES-GCM |
| Signing | PS256, PS384, PS512, RS256, ES256 |

## Security Best Practices

1. Never use the local file-based component in production
2. Use separate keys for encryption and signing
3. Enable key rotation in your vault and let Dapr handle re-encryption transparently
4. Scope the crypto component to only the services that need it

## Summary

The Dapr Cryptography API abstracts key management away from application code by routing all cryptographic operations through a managed vault backend. Applications only see plaintext and ciphertext - the keys themselves never leave the vault. This reduces the blast radius of application-layer security vulnerabilities and simplifies compliance with data protection regulations.
