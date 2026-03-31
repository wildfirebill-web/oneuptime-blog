# How to Implement Session Encryption in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Session, Security

Description: Encrypt session data before storing it in Redis using AES-GCM to protect sensitive user information from unauthorized access.

---

Storing session data in Redis in plain JSON means anyone with Redis access can read user information. Encrypting session data at the application layer adds a critical security layer - even if Redis is compromised, session contents remain unreadable without the encryption key.

## Encryption Strategy

Use AES-256-GCM (authenticated encryption) to encrypt session data before writing to Redis. AES-GCM provides both confidentiality and integrity, detecting tampering.

## Python Implementation with cryptography Library

```python
import redis
import json
import uuid
import base64
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Load key from environment - never hardcode
SESSION_KEY = bytes.fromhex(os.environ['SESSION_ENCRYPTION_KEY'])  # 32 bytes = 256-bit key
SESSION_TTL = 3600

def encrypt_session(data: dict) -> str:
    aesgcm = AESGCM(SESSION_KEY)
    nonce = os.urandom(12)  # 96-bit nonce for GCM
    plaintext = json.dumps(data).encode()
    ciphertext = aesgcm.encrypt(nonce, plaintext, None)
    # Combine nonce + ciphertext and base64 encode for storage
    return base64.b64encode(nonce + ciphertext).decode()

def decrypt_session(encrypted: str) -> dict:
    aesgcm = AESGCM(SESSION_KEY)
    raw = base64.b64decode(encrypted)
    nonce = raw[:12]
    ciphertext = raw[12:]
    plaintext = aesgcm.decrypt(nonce, ciphertext, None)
    return json.loads(plaintext)
```

## Creating and Reading Encrypted Sessions

```python
def create_session(user_id: str, data: dict) -> str:
    session_id = str(uuid.uuid4())
    session_data = {"user_id": user_id, **data}
    encrypted = encrypt_session(session_data)
    r.setex(f"session:{session_id}", SESSION_TTL, encrypted)
    return session_id

def get_session(session_id: str) -> dict | None:
    encrypted = r.get(f"session:{session_id}")
    if not encrypted:
        return None
    try:
        return decrypt_session(encrypted)
    except Exception:
        # Decryption failure - invalid or tampered session
        r.delete(f"session:{session_id}")
        return None
```

## Generating a Strong Encryption Key

```python
import secrets

# Generate a 256-bit (32-byte) key and print as hex
key = secrets.token_bytes(32)
print(key.hex())
```

Store the hex-encoded key in your environment as `SESSION_ENCRYPTION_KEY`.

## Key Rotation Strategy

When rotating encryption keys, decrypt old sessions with the old key and re-encrypt with the new key:

```python
OLD_KEY = bytes.fromhex(os.environ.get('SESSION_ENCRYPTION_KEY_OLD', ''))
NEW_KEY = bytes.fromhex(os.environ['SESSION_ENCRYPTION_KEY'])

def rotate_session(session_id: str):
    encrypted = r.get(f"session:{session_id}")
    if not encrypted:
        return

    # Try decrypting with old key
    try:
        aesgcm_old = AESGCM(OLD_KEY)
        raw = base64.b64decode(encrypted)
        plaintext = aesgcm_old.decrypt(raw[:12], raw[12:], None)
        data = json.loads(plaintext)
    except Exception:
        return  # Already encrypted with new key or invalid

    # Re-encrypt with new key
    new_encrypted = encrypt_session(data)
    r.setex(f"session:{session_id}", SESSION_TTL, new_encrypted)
```

## Verifying Encryption in Redis CLI

```bash
# Confirm the stored value is not readable plain JSON
redis-cli GET "session:some-session-id"
# Output: "7h2kLmN9Pq..."  (base64-encoded ciphertext, not plain JSON)
```

## Summary

Application-layer AES-256-GCM encryption ensures Redis session data is unreadable without the encryption key, even if the Redis instance is compromised. Always store encryption keys in environment variables or a secrets manager, use a fresh random nonce per encryption, and build a key rotation process to handle key lifecycle management over time.
