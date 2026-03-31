# How to Use Column-Level Encryption Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Encryption, Security, AES, Function, PII

Description: Learn how to use ClickHouse's built-in AES encryption functions to encrypt and decrypt column values for protecting sensitive data at the application level.

---

ClickHouse provides built-in functions for AES encryption and decryption, enabling application-level column encryption. This complements storage encryption by protecting specific sensitive fields even from privileged database users.

## Available Encryption Functions

ClickHouse includes these encryption functions:

- `encrypt('mode', plaintext, key)` - Encrypts data
- `decrypt('mode', ciphertext, key)` - Decrypts data
- `aes_encrypt_mysql('mode', plaintext, key)` - MySQL-compatible encryption
- `aes_decrypt_mysql('mode', ciphertext, key)` - MySQL-compatible decryption

Supported modes include `aes-128-ecb`, `aes-256-ecb`, `aes-128-cbc`, `aes-256-cbc`, and `aes-256-gcm`.

## Basic Encryption Example

```sql
-- Encrypt a value
SELECT encrypt('aes-256-cbc', 'sensitive_data_here', '0123456789abcdef0123456789abcdef');
```

## Creating a Table with Encrypted Columns

Store encrypted PII in binary columns:

```sql
CREATE TABLE user_profiles (
    user_id UInt64,
    username String,
    -- Store encrypted values as binary strings
    email_encrypted String,
    phone_encrypted String,
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY (user_id, created_at);
```

## Inserting Encrypted Data

```sql
INSERT INTO user_profiles (user_id, username, email_encrypted, phone_encrypted, created_at)
VALUES (
    1001,
    'johndoe',
    encrypt('aes-256-gcm', 'john@example.com', '0123456789abcdef0123456789abcdef', 'initialization_v'),
    encrypt('aes-256-gcm', '+1-555-0100', '0123456789abcdef0123456789abcdef', 'initialization_v'),
    now()
);
```

## Querying Encrypted Data

Decrypt at query time for authorized users:

```sql
SELECT
    user_id,
    username,
    decrypt('aes-256-gcm',
        email_encrypted,
        '0123456789abcdef0123456789abcdef',
        'initialization_v') AS email,
    decrypt('aes-256-gcm',
        phone_encrypted,
        '0123456789abcdef0123456789abcdef',
        'initialization_v') AS phone
FROM user_profiles
WHERE user_id = 1001;
```

Unauthorized users who can SELECT the table only see ciphertext:

```sql
-- Without the key, data is unreadable
SELECT email_encrypted FROM user_profiles LIMIT 5;
```

## Searching Encrypted Data

You cannot efficiently search encrypted data directly. Use a hash of the value for lookup:

```sql
CREATE TABLE user_profiles (
    user_id UInt64,
    username String,
    email_encrypted String,
    -- Hash for searching without exposing plaintext
    email_hash String,
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY (email_hash, user_id);

-- Insert with hash
INSERT INTO user_profiles VALUES (
    1001, 'johndoe',
    encrypt('aes-256-gcm', 'john@example.com', 'key_32_bytes_long_here_pad_it_ok', 'iv_16bytes_long_'),
    SHA256('john@example.com'),
    now()
);

-- Search by hash
SELECT user_id, username
FROM user_profiles
WHERE email_hash = SHA256('john@example.com');
```

## Using a Secure Key Store

Never hardcode encryption keys in queries. Retrieve them from a secrets manager:

```sql
-- Use a ClickHouse secret function or environment variable approach
-- Keys should be managed externally and passed at query time
SELECT decrypt(
    'aes-256-gcm',
    email_encrypted,
    dictGet('key_store', 'key_value', 'email_key'),
    dictGet('key_store', 'iv_value', 'email_key')
) AS email
FROM user_profiles;
```

## Best Practices

- Use AES-256-GCM for authenticated encryption (prevents tampering)
- Rotate encryption keys periodically and re-encrypt affected rows
- Store encryption keys separately from the database
- Grant `decrypt` query execution only to privileged roles
- Combine with column-level GRANT restrictions for defense in depth

## Summary

ClickHouse's built-in encryption functions let you protect sensitive column values at the application level. Use AES-256-GCM for strong authenticated encryption, store ciphertext in String columns, and control decryption access through query-level permissions. This provides an additional security layer beyond storage-level encryption.
