# How to Use Column-Level Encryption Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Encryption, Security, Column Encryption, Data Protection

Description: Learn how to encrypt and decrypt specific columns in ClickHouse using built-in encryption functions to protect sensitive data at rest and in transit.

---

ClickHouse provides built-in encryption functions (`encrypt`, `decrypt`, `aes_encrypt_mysql`, `aes_decrypt_mysql`) for encrypting column values at the SQL level. This protects sensitive data like PII, credentials, or financial information while still allowing it to be stored and queried.

## Available Encryption Functions

ClickHouse supports several encryption functions:

| Function | Description |
|---|---|
| `encrypt('mode', data, key)` | Encrypts data using AES |
| `decrypt('mode', data, key)` | Decrypts data encrypted with encrypt() |
| `aes_encrypt_mysql('mode', data, key)` | MySQL-compatible AES encryption |
| `aes_decrypt_mysql('mode', data, key)` | MySQL-compatible AES decryption |

Supported modes include `aes-128-ecb`, `aes-256-cbc`, `aes-128-gcm`, `aes-256-gcm`.

## Basic Encryption Example

Encrypt a value when inserting:

```sql
INSERT INTO users (id, name, ssn_encrypted)
VALUES (
    1,
    'Alice Smith',
    encrypt('aes-256-gcm', '123-45-6789', '32bytekeymustbeexactly32byteslng', 'initialization_v')
);
```

Decrypt when reading:

```sql
SELECT
    id,
    name,
    decrypt('aes-256-gcm', ssn_encrypted, '32bytekeymustbeexactly32byteslng', 'initialization_v') AS ssn
FROM users;
```

## Creating a Table with Encrypted Columns

Design a table where sensitive columns store encrypted binary data:

```sql
CREATE TABLE customer_pii (
    customer_id UInt64,
    name String,
    email_encrypted String,
    phone_encrypted String,
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY (customer_id, created_at);
```

Insert with encryption:

```sql
INSERT INTO customer_pii
SELECT
    customer_id,
    name,
    encrypt('aes-256-gcm', email, '32bytekeymustbeexactly32byteslng', 'nonce12345678901') AS email_encrypted,
    encrypt('aes-256-gcm', phone, '32bytekeymustbeexactly32byteslng', 'nonce12345678901') AS phone_encrypted,
    now()
FROM raw_customer_import;
```

## Using AES-GCM for Authenticated Encryption

AES-GCM provides both encryption and authentication (tamper detection):

```sql
-- Encrypt with AES-256-GCM (recommended)
SELECT encrypt(
    'aes-256-gcm',
    'sensitive data here',
    '32bytekeyexactlythirtytwocharss!',  -- 32-byte key for AES-256
    'uniquenonce1234567'                  -- 12-byte nonce (initialization vector)
) AS encrypted_value;
```

## Storing the Encryption Key Securely

Never hardcode encryption keys in queries. Use ClickHouse's `named_collections` or environment variables:

```xml
<!-- config.d/encryption_keys.xml -->
<clickhouse>
  <named_collections>
    <encryption>
      <pii_key>32bytekeymustbeexactly32byteslng</pii_key>
    </encryption>
  </named_collections>
</clickhouse>
```

Reference in queries:

```sql
SELECT
    decrypt(
        'aes-256-gcm',
        email_encrypted,
        (SELECT pii_key FROM system.named_collections WHERE collection = 'encryption'),
        'nonce12345678901'
    ) AS email
FROM customer_pii
WHERE customer_id = 12345;
```

## Using Column Codecs Instead of SQL Encryption

For transparent column-level encryption without SQL overhead, use column codecs:

```sql
CREATE TABLE secure_events (
    id UInt64,
    payload String CODEC(AES_128_GCM_SIV)
) ENGINE = MergeTree()
ORDER BY id;
```

This encrypts data at the storage level transparently, requiring the key in server configuration:

```xml
<encryption_codecs>
  <aes_128_gcm_siv>
    <key_hex>0123456789abcdef0123456789abcdef</key_hex>
  </aes_128_gcm_siv>
</encryption_codecs>
```

## Decrypting Data for Analytics

For analytics on encrypted PII, decrypt at query time using a view:

```sql
CREATE VIEW customer_decrypted AS
SELECT
    customer_id,
    name,
    decrypt('aes-256-gcm', email_encrypted, '...key...', '...nonce...') AS email,
    decrypt('aes-256-gcm', phone_encrypted, '...key...', '...nonce...') AS phone,
    created_at
FROM customer_pii;
```

Grant access to the view only to users who should see decrypted data.

## Summary

ClickHouse column-level encryption uses `encrypt()` and `decrypt()` functions or transparent storage codecs to protect sensitive column values. Use AES-256-GCM for authenticated encryption, store keys in named collections rather than hardcoding them, and create views for role-based access to decrypted data. For performance-sensitive workloads, prefer AES column codecs over SQL-level encryption functions.
