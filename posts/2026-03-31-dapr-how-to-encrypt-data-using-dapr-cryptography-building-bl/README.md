# How to Encrypt Data Using Dapr Cryptography Building Block

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cryptography, Encryption, Security, Data Protection

Description: Learn how to use the Dapr Cryptography building block to encrypt and decrypt data in your microservices without managing cryptographic keys directly in code.

---

## Overview

The Dapr Cryptography building block provides a simple API for encrypting and decrypting data without exposing cryptographic keys to your application. Keys are managed by an external provider such as Azure Key Vault, HashiCorp Vault, or a local key file, while Dapr handles the encryption operations through the sidecar.

## Supported Key Providers

- Azure Key Vault
- HashiCorp Vault
- Local file-based keys (development only)
- Kubernetes secrets

## Configure a Cryptography Component

### Local Key File (Development)

Generate a key file:

```bash
openssl rand -base64 32 > /tmp/mykey.key
```

Define the component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: localstorage-crypto
  namespace: default
spec:
  type: crypto.dapr.localstorage
  version: v1
  metadata:
    - name: path
      value: "/tmp"
```

### Azure Key Vault

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault-crypto
  namespace: default
spec:
  type: crypto.dapr.azure.keyvault
  version: v1
  metadata:
    - name: vaultName
      value: "my-crypto-vault"
    - name: azureTenantId
      value: "your-tenant-id"
    - name: azureClientId
      value: "your-client-id"
    - name: azureClientSecret
      value: "your-client-secret"
```

Apply the component:

```bash
kubectl apply -f crypto-component.yaml
```

## Encrypting Data via the HTTP API

Encrypt a string payload:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/crypto/localstorage-crypto/encrypt \
  -H "Content-Type: application/json" \
  -d '{
    "plaintext": "sensitive customer data",
    "keyName": "mykey",
    "algorithm": "A256GCM"
  }'
```

Response:

```json
{
  "ciphertext": "eyJhbGciOiJBMjU2R0NNS1ciLCJlbmMiOiJBMjU2R0NNIn0..."
}
```

Decrypt the ciphertext:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/crypto/localstorage-crypto/decrypt \
  -H "Content-Type: application/json" \
  -d '{
    "ciphertext": "eyJhbGciOiJBMjU2R0NNS1ciLCJlbmMiOiJBMjU2R0NNIn0...",
    "keyName": "mykey"
  }'
```

## Encrypting Data in .NET

```csharp
using Dapr.Client;
using System.Text;

var client = new DaprClientBuilder().Build();

// Encrypt
var plaintext = Encoding.UTF8.GetBytes("Sensitive PII data: user@example.com");

await using var encryptStream = await client.EncryptAsync(
    "localstorage-crypto",
    new MemoryStream(plaintext),
    "mykey",
    new EncryptionOptions(KeyWrapAlgorithm.AesGcm)
    {
        DataEncryptionKeyName = "mykey",
        EncryptionAlgorithm = "A256GCM"
    }
);

using var memStream = new MemoryStream();
await encryptStream.CopyToAsync(memStream);
var ciphertext = memStream.ToArray();
Console.WriteLine($"Encrypted {ciphertext.Length} bytes");

// Decrypt
await using var decryptStream = await client.DecryptAsync(
    "localstorage-crypto",
    new MemoryStream(ciphertext),
    "mykey",
    new DecryptionOptions()
);

using var resultStream = new MemoryStream();
await decryptStream.CopyToAsync(resultStream);
var decrypted = Encoding.UTF8.GetString(resultStream.ToArray());
Console.WriteLine($"Decrypted: {decrypted}");
```

## Encrypting Data in Python

```python
import asyncio
from dapr.clients import DaprClient
from dapr.clients.grpc._crypto import EncryptOptions, DecryptOptions

async def encrypt_decrypt_example():
    with DaprClient() as client:
        plaintext = b"Sensitive customer data"

        # Encrypt
        encrypt_options = EncryptOptions(
            component_name="localstorage-crypto",
            key_name="mykey",
            key_wrap_algorithm="AES",
        )

        ciphertext = b""
        async for chunk in client.encrypt(iter([plaintext]), encrypt_options):
            ciphertext += chunk

        print(f"Encrypted {len(ciphertext)} bytes")

        # Decrypt
        decrypt_options = DecryptOptions(
            component_name="localstorage-crypto",
            key_name="mykey",
        )

        plaintext_out = b""
        async for chunk in client.decrypt(iter([ciphertext]), decrypt_options):
            plaintext_out += chunk

        print(f"Decrypted: {plaintext_out.decode()}")

asyncio.run(encrypt_decrypt_example())
```

## Encrypting Files

Encrypt a file using streaming:

```csharp
using var inputFile = File.OpenRead("/tmp/sensitive-report.pdf");
using var outputFile = File.Create("/tmp/sensitive-report.pdf.enc");

await using var encryptStream = await client.EncryptAsync(
    "azurekeyvault-crypto",
    inputFile,
    "report-encryption-key",
    new EncryptionOptions(KeyWrapAlgorithm.RsaOaep256)
);

await encryptStream.CopyToAsync(outputFile);
Console.WriteLine("File encrypted successfully");
```

Decrypt the file later:

```csharp
using var encryptedFile = File.OpenRead("/tmp/sensitive-report.pdf.enc");
using var outputFile = File.Create("/tmp/sensitive-report-decrypted.pdf");

await using var decryptStream = await client.DecryptAsync(
    "azurekeyvault-crypto",
    encryptedFile,
    "report-encryption-key",
    new DecryptionOptions()
);

await decryptStream.CopyToAsync(outputFile);
Console.WriteLine("File decrypted successfully");
```

## Supported Algorithms

| Algorithm | Use Case |
|-----------|---------|
| A256GCM | Symmetric encryption for data at rest |
| A128CBC-HS256 | Symmetric encryption with HMAC |
| RSA-OAEP-256 | Asymmetric encryption for key wrapping |

## Dapr CLI Test

Test encryption locally using the Dapr HTTP API:

```bash
dapr run --app-id crypto-test --resources-path ./components -- python3 app.py
```

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/crypto/localstorage-crypto/encrypt \
  -H "Content-Type: application/json" \
  -d '{"plaintext": "hello world", "keyName": "mykey", "algorithm": "A256GCM"}'
```

## Summary

The Dapr Cryptography building block provides encrypt and decrypt operations through the Dapr sidecar, keeping cryptographic keys out of application code entirely. You configure a key provider component (Azure Key Vault, HashiCorp Vault, or local files), and your application calls the Dapr API to encrypt or decrypt data using named keys. Streaming support makes it practical to encrypt large files without loading them fully into memory.
