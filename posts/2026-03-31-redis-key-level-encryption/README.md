# How to Implement Redis Key-Level Encryption

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Encryption, Cryptography, Privacy

Description: Implement application-side key-level encryption for Redis values to protect sensitive data at rest, even if the Redis server or its backups are compromised.

---

Redis does not natively encrypt data at rest. If an attacker gains access to your Redis memory, RDB snapshots, or AOF files, all stored data is readable. Key-level encryption at the application layer encrypts values before writing to Redis, so even a compromised server reveals only ciphertext.

## Choosing an Encryption Algorithm

Use AES-256-GCM (authenticated encryption). It provides both confidentiality and integrity, detecting tampering without a separate HMAC step.

```bash
pip install cryptography
```

## Building an Encrypted Redis Client Wrapper

```python
import os
import base64
import redis
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

class EncryptedRedis:
    def __init__(self, host: str, port: int, key: bytes):
        self.client = redis.Redis(host=host, port=port, decode_responses=False)
        if len(key) != 32:
            raise ValueError("Key must be 32 bytes for AES-256")
        self.aesgcm = AESGCM(key)

    def _encrypt(self, plaintext: str) -> bytes:
        nonce = os.urandom(12)  # 96-bit nonce for GCM
        ciphertext = self.aesgcm.encrypt(nonce, plaintext.encode(), None)
        return base64.b64encode(nonce + ciphertext)

    def _decrypt(self, data: bytes) -> str:
        raw = base64.b64decode(data)
        nonce = raw[:12]
        ciphertext = raw[12:]
        plaintext = self.aesgcm.decrypt(nonce, ciphertext, None)
        return plaintext.decode()

    def set(self, key: str, value: str, ex: int = None):
        encrypted = self._encrypt(value)
        self.client.set(key, encrypted, ex=ex)

    def get(self, key: str) -> str:
        data = self.client.get(key)
        if data is None:
            return None
        return self._decrypt(data)

    def delete(self, key: str):
        self.client.delete(key)
```

## Usage Example

```python
import secrets

# Generate a 32-byte key (store securely, e.g., in HashiCorp Vault)
encryption_key = secrets.token_bytes(32)

enc_redis = EncryptedRedis("localhost", 6379, encryption_key)

# Encrypt and store
enc_redis.set("user:42:ssn", "123-45-6789", ex=3600)
enc_redis.set("user:42:credit_card", "4111111111111111", ex=3600)

# Decrypt and retrieve
ssn = enc_redis.get("user:42:ssn")
print(ssn)  # 123-45-6789

# Raw Redis value is ciphertext
raw = enc_redis.client.get("user:42:ssn")
print(raw)  # b'AjR...encrypted...'
```

## Key Management Best Practices

Never hardcode the encryption key. Load it from an environment variable or a secrets manager:

```python
import os

key_hex = os.environ.get("REDIS_ENCRYPTION_KEY")
if not key_hex:
    raise RuntimeError("REDIS_ENCRYPTION_KEY not set")
encryption_key = bytes.fromhex(key_hex)
```

Generate and store the key:

```bash
python3 -c "import secrets; print(secrets.token_bytes(32).hex())"
# Store output in your secrets manager
```

## Encrypting Hashes

For Redis Hashes, encrypt each field value individually:

```python
def hset_encrypted(self, name: str, mapping: dict):
    encrypted_mapping = {k: self._encrypt(v) for k, v in mapping.items()}
    self.client.hset(name, mapping=encrypted_mapping)

def hgetall_decrypted(self, name: str) -> dict:
    raw = self.client.hgetall(name)
    return {k.decode(): self._decrypt(v) for k, v in raw.items()}
```

## What This Does Not Protect

Key-level encryption protects values at rest and in RDB/AOF files, but it does not protect:
- Key names (keys are stored in plaintext)
- Metadata like TTLs and key counts
- Data in transit (use TLS for that)

## Summary

Application-side key-level encryption using AES-256-GCM encrypts Redis values before storage, ensuring that RDB snapshots, AOF files, and direct memory access expose only ciphertext. Manage encryption keys through a secrets manager and rotate them periodically. This approach adds meaningful protection for sensitive fields like PII, tokens, and credentials stored in Redis.
