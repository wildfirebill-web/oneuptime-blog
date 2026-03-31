# How to Encrypt Data Using Dapr Cryptography Building Block

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cryptography, Encryption, Security, Microservice

Description: Learn how to use the Dapr Cryptography building block to encrypt and decrypt data in your microservices without managing cryptographic keys or libraries directly.

---

## What Is the Dapr Cryptography Building Block

The Dapr Cryptography building block provides encryption and decryption capabilities through the Dapr sidecar, abstracting away key management and cryptographic implementation details. Your application calls the Dapr API to encrypt or decrypt data, and Dapr handles key retrieval, algorithm selection, and the actual cryptographic operations using the configured key store.

## Prerequisites

- Dapr CLI installed and initialized (v1.11+)
- A key store configured (local file, Azure Key Vault, Kubernetes secrets)
- Basic familiarity with Dapr components

## Define a Local Key Store for Development

For local development, use a JSON-based key store:

```bash
# Create the key configuration file
mkdir -p ~/.dapr/certs
cat > ~/.dapr/certs/local-keys.json << 'EOF'
{
  "version": "1",
  "keys": [
    {
      "name": "mykey",
      "type": "AES-CBC",
      "size": 256,
      "key": "YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXoxMjM0NTY="
    }
  ]
}
EOF
```

Define the cryptography component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: local-crypto
  namespace: default
spec:
  type: crypto.dapr.localstorage
  version: v1
  metadata:
  - name: path
    value: "/home/user/.dapr/certs"
```

## Use Azure Key Vault for Production Keys

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekv-crypto
  namespace: default
spec:
  type: crypto.azure.keyvault
  version: v1
  metadata:
  - name: vaultName
    value: "my-key-vault"
  - name: azureTenantId
    value: "your-tenant-id"
  - name: azureClientId
    value: "your-client-id"
  - name: azureClientSecret
    secretKeyRef:
      name: azure-creds
      key: clientSecret
```

## Encrypt Data via the HTTP API

```bash
# Encrypt a payload
curl -X PUT \
  "http://localhost:3500/v1.0-alpha1/crypto/local-crypto/encrypt" \
  -H "dapr-key-name: mykey" \
  -H "dapr-key-wrap-algorithm: A256KW" \
  -H "Content-Type: application/octet-stream" \
  --data-binary "sensitive data to encrypt"
```

The response is the encrypted ciphertext in binary form.

## Decrypt Data via the HTTP API

```bash
curl -X PUT \
  "http://localhost:3500/v1.0-alpha1/crypto/local-crypto/decrypt" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @encrypted-data.bin
```

## Use in a Node.js Application

```javascript
const { DaprClient, CryptoClient } = require('@dapr/dapr');
const { Buffer } = require('buffer');

const client = new DaprClient();
const CRYPTO_COMPONENT = 'local-crypto';
const KEY_NAME = 'mykey';

async function encryptData(plaintext) {
  const plaintextBuffer = Buffer.from(plaintext, 'utf8');

  const encrypted = await client.crypto.encrypt(
    CRYPTO_COMPONENT,
    plaintextBuffer,
    {
      keyName: KEY_NAME,
      keyWrapAlgorithm: 'A256KW',
    }
  );

  return encrypted.toString('base64');
}

async function decryptData(encryptedBase64) {
  const encryptedBuffer = Buffer.from(encryptedBase64, 'base64');

  const decrypted = await client.crypto.decrypt(
    CRYPTO_COMPONENT,
    encryptedBuffer
  );

  return decrypted.toString('utf8');
}

// Example: encrypt a PII field before storing
async function saveUserWithEncryptedSSN(user) {
  const encryptedSSN = await encryptData(user.ssn);

  await db.users.insert({
    id: user.id,
    name: user.name,
    email: user.email,
    ssn_encrypted: encryptedSSN, // store encrypted
  });

  console.log('User saved with encrypted SSN');
}

// Example: decrypt when needed
async function getUserSSN(userId) {
  const user = await db.users.findById(userId);
  return await decryptData(user.ssn_encrypted);
}
```

## Use in Python

```python
from dapr.clients import DaprClient
import base64

CRYPTO_COMPONENT = 'local-crypto'
KEY_NAME = 'mykey'

def encrypt_data(plaintext: str) -> str:
    with DaprClient() as client:
        plaintext_bytes = plaintext.encode('utf-8')
        encrypted = client.encrypt(
            data=plaintext_bytes,
            options={
                'componentName': CRYPTO_COMPONENT,
                'keyName': KEY_NAME,
                'keyWrapAlgorithm': 'A256KW',
            }
        )
        return base64.b64encode(encrypted).decode('utf-8')

def decrypt_data(encrypted_b64: str) -> str:
    with DaprClient() as client:
        encrypted_bytes = base64.b64decode(encrypted_b64)
        decrypted = client.decrypt(
            data=encrypted_bytes,
            options={'componentName': CRYPTO_COMPONENT}
        )
        return decrypted.decode('utf-8')

# Encrypt credit card number before storage
def store_payment_method(user_id: str, card_number: str):
    encrypted_card = encrypt_data(card_number)
    db.payment_methods.insert({'user_id': user_id, 'card_encrypted': encrypted_card})
    print("Payment method stored securely")
```

## Supported Algorithms

```text
Encryption:
  AES-CBC        - symmetric, fast for large data
  AES-GCM        - symmetric with authentication
  ChaCha20-Poly1305 - modern symmetric cipher

Key Wrapping:
  A256KW         - AES key wrap, 256-bit key
  RSA-OAEP-256   - asymmetric key wrapping with SHA-256

Signing (available in some providers):
  RS256, RS384, RS512  - RSA signatures
  ES256, ES384, ES512  - ECDSA signatures
```

## Summary

The Dapr Cryptography building block provides a secure, abstracted API for encrypting and decrypting data without embedding cryptographic libraries or managing keys in application code. By delegating key management to providers like Azure Key Vault or Kubernetes secrets, and performing crypto operations through the Dapr sidecar, microservices can handle sensitive data encryption with minimal code and maximum portability across environments.
