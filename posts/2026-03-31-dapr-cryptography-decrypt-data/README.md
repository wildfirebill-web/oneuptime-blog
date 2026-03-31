# How to Decrypt Data Using Dapr Cryptography Building Block

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cryptography, Decryption, Security, AES, Key Management

Description: Learn how to decrypt data using the Dapr Cryptography building block, with examples in Go, Python, and the HTTP API using AES-GCM and RSA key wrapping.

---

## Decryption with Dapr Cryptography

The Dapr Cryptography building block handles decryption by:
1. Receiving your ciphertext
2. Fetching the decryption key from the configured provider
3. Unwrapping the data encryption key
4. Decrypting the data using AES-256-GCM
5. Returning the plaintext

Your application never directly handles private keys. Dapr retrieves them from the key provider at runtime.

## Prerequisites

You need a configured cryptography component pointing to the key that was used for encryption.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-crypto
spec:
  type: crypto.dapr.localstorage
  version: v1
  metadata:
  - name: path
    value: ./keys
```

## HTTP API Decryption

```bash
# Decrypt base64-encoded ciphertext
curl -X POST http://localhost:3500/v1.0-alpha1/crypto/my-crypto/decrypt \
  -H "Content-Type: application/octet-stream" \
  -H "dapr-key-name: customer-data-key" \
  --data-binary @ciphertext.bin \
  -o plaintext.txt
```

## Go SDK Decryption

```go
package main

import (
    "bytes"
    "context"
    "io"
    "log"
    dapr "github.com/dapr/go-sdk/client"
)

func decryptData(ciphertext []byte) ([]byte, error) {
    client, err := dapr.NewClient()
    if err != nil {
        return nil, err
    }
    defer client.Close()

    decryptOpts := dapr.DecryptRequestOptions{
        ComponentName: "my-crypto",
        KeyName:       "customer-data-key",
    }

    decryptedStream, err := client.Decrypt(
        context.Background(),
        bytes.NewReader(ciphertext),
        decryptOpts,
    )
    if err != nil {
        return nil, err
    }

    return io.ReadAll(decryptedStream)
}

func main() {
    // Read ciphertext from database
    ciphertext := readCiphertextFromDB("record-123")

    plaintext, err := decryptData(ciphertext)
    if err != nil {
        log.Fatalf("Decryption failed: %v", err)
    }

    log.Printf("Decrypted: %s", string(plaintext))
}
```

## Python SDK Decryption

```python
import io
from dapr.clients import DaprClient

def decrypt_field(ciphertext: bytes, key_name: str) -> bytes:
    with DaprClient() as d:
        decrypted_stream = d.decrypt(
            data=io.BytesIO(ciphertext),
            options={
                "componentName": "my-crypto",
                "keyName": key_name
            }
        )
        return decrypted_stream.read()

# Example: decrypt PII fields from database records
def get_customer_record(customer_id: str) -> dict:
    row = db.query("SELECT * FROM customers WHERE id = %s", customer_id)
    return {
        "id": row["id"],
        "name": decrypt_field(row["name_encrypted"], "pii-key").decode(),
        "ssn": decrypt_field(row["ssn_encrypted"], "pii-key").decode(),
        "email": row["email"]  # not encrypted
    }
```

## JavaScript SDK Decryption

```javascript
const { DaprClient } = require("@dapr/dapr");
const { Readable } = require("stream");

const client = new DaprClient();

async function decryptData(ciphertext) {
    const stream = Readable.from([ciphertext]);

    const decryptedStream = await client.crypto.decrypt(stream, {
        componentName: "my-crypto",
        keyName: "customer-data-key"
    });

    const chunks = [];
    for await (const chunk of decryptedStream) {
        chunks.push(chunk);
    }
    return Buffer.concat(chunks);
}

// Usage
const ciphertext = Buffer.from(encryptedBase64, "base64");
const plaintext = await decryptData(ciphertext);
console.log("Decrypted:", plaintext.toString());
```

## Handling Decryption Errors

```go
decryptedStream, err := client.Decrypt(ctx, reader, opts)
if err != nil {
    if strings.Contains(err.Error(), "key not found") {
        // Key was rotated or deleted
        return nil, fmt.Errorf("decryption key unavailable: %w", err)
    }
    if strings.Contains(err.Error(), "authentication failed") {
        // Ciphertext was tampered with
        return nil, fmt.Errorf("data integrity check failed: %w", err)
    }
    return nil, err
}
```

## Batch Decryption Pattern

```python
def decrypt_records_batch(records: list) -> list:
    with DaprClient() as d:
        decrypted = []
        for record in records:
            plain = d.decrypt(
                data=io.BytesIO(record["ciphertext"]),
                options={"componentName": "my-crypto", "keyName": "data-key"}
            ).read()
            record["plaintext"] = plain.decode()
            decrypted.append(record)
    return decrypted
```

## Summary

The Dapr Cryptography building block makes decryption straightforward: provide the ciphertext and key name, and Dapr handles key retrieval, unwrapping, and AES-256-GCM decryption. The same code works against any key provider (local, Azure Key Vault, Kubernetes), making it easy to upgrade key management without changing application code.
