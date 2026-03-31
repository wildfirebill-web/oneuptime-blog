# How to Use Dapr Cryptography with Local Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Cryptography, Local Storage, Development, Key Management, Encryption

Description: Learn how to configure Dapr Cryptography with local file-based key storage for development and testing, enabling encryption without external key vault dependencies.

---

## Local Storage for Development Cryptography

The Dapr local storage cryptography provider reads keys from the local filesystem. It is designed for development and testing environments where you want to exercise the Dapr Cryptography API without configuring a cloud key vault or Kubernetes secrets. In production, swap this component for Azure Key Vault or Kubernetes secrets.

## Generating Keys for Local Storage

Keys must be in JWK (JSON Web Key) format. You can generate them with OpenSSL and convert them, or use the Dapr CLI:

```bash
# Create keys directory
mkdir -p ./keys

# Generate AES-256 key in JWK format
cat > ./keys/aes-key.json << 'EOF'
{
  "kty": "oct",
  "kid": "aes-256-key",
  "use": "enc",
  "alg": "A256KW",
  "k": "$(openssl rand -base64 32)"
}
EOF
```

Or generate an RSA key pair:

```bash
# Generate RSA key pair (private key includes public key)
openssl genrsa -out ./keys/rsa-private.pem 4096

# Convert RSA private key to JWK format
cat > convert-key.js << 'EOF'
const fs = require("fs");
const crypto = require("crypto");
const pem = fs.readFileSync("./keys/rsa-private.pem");
const keyObject = crypto.createPrivateKey(pem);
const jwk = keyObject.export({ format: "jwk" });
jwk.kid = "rsa-key-v1";
jwk.use = "enc";
fs.writeFileSync("./keys/rsa-key.json", JSON.stringify(jwk, null, 2));
console.log("Key written to ./keys/rsa-key.json");
EOF
node convert-key.js
```

## Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: local-crypto
spec:
  type: crypto.dapr.localstorage
  version: v1
  metadata:
  - name: path
    value: ./keys
```

The `path` is relative to the Dapr component directory or can be an absolute path.

## Encrypting Data with Local Keys

```python
import io
from dapr.clients import DaprClient

def encrypt_data(plaintext: bytes, key_name: str = "aes-256-key") -> bytes:
    with DaprClient() as d:
        encrypted = d.encrypt(
            data=io.BytesIO(plaintext),
            options={
                "componentName": "local-crypto",
                "keyName": key_name,
                "keyWrapAlgorithm": "AES"
            }
        )
        return encrypted.read()

# Test encryption locally
message = b"This is a secret message for local testing"
ciphertext = encrypt_data(message)
print(f"Encrypted {len(message)} bytes -> {len(ciphertext)} bytes")
```

## Decrypting Data

```python
def decrypt_data(ciphertext: bytes, key_name: str = "aes-256-key") -> bytes:
    with DaprClient() as d:
        decrypted = d.decrypt(
            data=io.BytesIO(ciphertext),
            options={
                "componentName": "local-crypto",
                "keyName": key_name
            }
        )
        return decrypted.read()

plaintext = decrypt_data(ciphertext)
print(f"Decrypted: {plaintext.decode()}")
```

## Running the App with Local Crypto

```bash
dapr run \
  --app-id my-app \
  --app-port 6000 \
  --resources-path ./components \
  -- python app.py
```

The `./components` directory should contain the `local-crypto` component YAML.

## Organizing Keys by Environment

```text
keys/
  dev/
    aes-256-key.json
    rsa-key.json
  staging/
    aes-256-key.json
    rsa-key.json
```

```yaml
# components/dev/crypto.yaml
spec:
  type: crypto.dapr.localstorage
  version: v1
  metadata:
  - name: path
    value: ./keys/dev
```

## Multiple Keys in a Single Directory

The local provider reads individual `.json` files from the directory. Each file's base name (without `.json`) is the key name:

```text
keys/
  customer-data-key.json    # keyName: "customer-data-key"
  payment-key.json          # keyName: "payment-key"
  signing-key.json          # keyName: "signing-key"
```

```python
# Use different keys for different data types
encrypted_ssn = encrypt_data(ssn_bytes, key_name="customer-data-key")
encrypted_card = encrypt_data(card_bytes, key_name="payment-key")
```

## Switching to Production Key Provider

To switch from local storage to Azure Key Vault, only the component YAML changes:

```yaml
# components/production/crypto.yaml
spec:
  type: crypto.azure.keyvault
  version: v1
  metadata:
  - name: vaultURI
    value: "https://my-vault.vault.azure.net"
  - name: azureClientId
    value: "<managed-identity-id>"
```

Application code remains identical - just the component name and configuration change.

## Summary

The Dapr local storage cryptography provider is ideal for development and testing: it requires no external services, keys are simple JSON files, and the same application code works in production by swapping the component YAML to point at Azure Key Vault or Kubernetes Secrets. This makes the Cryptography API a portable abstraction across all environments.
