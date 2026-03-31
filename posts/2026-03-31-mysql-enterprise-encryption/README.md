# How to Use MySQL Enterprise Encryption

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Enterprise, Encryption, Security

Description: Learn how to use MySQL Enterprise Encryption functions to perform RSA and AES key operations, encrypt data, and create digital signatures within SQL queries.

---

## What Is MySQL Enterprise Encryption?

MySQL Enterprise Encryption is a set of SQL functions available in MySQL Enterprise Edition that expose cryptographic operations - asymmetric key generation, encryption/decryption, digital signatures, and key derivation - directly within MySQL queries. In MySQL 8.0.28+, these functions are provided by the `component_enterprise_encryption` component.

## Installing the Component

```sql
-- MySQL 8.0.28+: install the component
INSTALL COMPONENT 'file://component_enterprise_encryption';

-- Verify
SELECT * FROM mysql.component WHERE component_urn LIKE '%enterprise_encryption%';
```

On older 8.0 versions, install the `openssl_udf` plugin:

```sql
INSTALL PLUGIN openssl_udf SONAME 'openssl_udf.so';
```

## Generating Asymmetric Keys

```sql
-- Generate a 2048-bit RSA private key
SET @private_key = create_asymmetric_priv_key('RSA', 2048);

-- Derive the public key from the private key
SET @public_key = create_asymmetric_pub_key('RSA', @private_key);

-- Display key lengths
SELECT LENGTH(@private_key) AS priv_key_length,
       LENGTH(@public_key)  AS pub_key_length;
```

Supported algorithms:

```text
RSA  - for encryption and signing
DSA  - for signing only (not encryption)
DH   - for key exchange (Diffie-Hellman)
```

## Encrypting and Decrypting Data

Asymmetric encryption in MySQL Enterprise Encryption uses RSA:

```sql
-- Encrypt a message with the public key (anyone can encrypt)
SET @plaintext = 'Sensitive Data 2026';
SET @ciphertext = asymmetric_encrypt('RSA', @plaintext, @public_key);

-- Decrypt with the private key (only the holder can decrypt)
SET @decrypted = asymmetric_decrypt('RSA', @ciphertext, @private_key);

SELECT @decrypted;  -- Should return: Sensitive Data 2026
```

## Digital Signatures

```sql
-- Sign a message digest with the private key
SET @message = 'order:12345:total:99.99';
SET @digest = create_digest('SHA256', @message);
SET @signature = asymmetric_sign('RSA', @digest, @private_key, 'SHA256');

-- Verify the signature with the public key
SELECT asymmetric_verify('RSA', @digest, @signature, @public_key, 'SHA256') AS valid;
-- Result: 1 (valid) or 0 (invalid)
```

## Generating Symmetric Keys for AES

For bulk data encryption, use AES with a derived key:

```sql
-- Derive a symmetric key using DH key exchange
SET @dh_params = create_dh_parameters(2048);
SET @dh_priv   = create_asymmetric_priv_key('DH', @dh_params);
SET @dh_pub    = create_asymmetric_pub_key('DH', @dh_priv);

-- Use AES_ENCRYPT with a strong key for column-level encryption
SET @aes_key = UNHEX(SHA2('my-secret-passphrase', 256));

-- Encrypt a value
SELECT HEX(AES_ENCRYPT('credit_card_number', @aes_key)) AS encrypted_value;

-- Decrypt a value
SELECT AES_DECRYPT(UNHEX('...encrypted_hex...'), @aes_key) AS decrypted_value;
```

## Storing Encrypted Data in a Table

```sql
CREATE TABLE payment_data (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT,
  encrypted_card VARBINARY(256)
);

-- Insert with encryption
INSERT INTO payment_data (user_id, encrypted_card)
VALUES (1, AES_ENCRYPT('4111111111111111', @aes_key));

-- Retrieve and decrypt
SELECT user_id, AES_DECRYPT(encrypted_card, @aes_key) AS card_number
FROM payment_data
WHERE user_id = 1;
```

## Summary

MySQL Enterprise Encryption exposes RSA, DSA, and DH cryptographic operations through SQL functions, letting you perform key generation, asymmetric encryption, and digital signing without leaving the database layer. For high-volume data encryption, combine it with AES functions using properly derived keys. Always store private keys and passphrases outside the database, in a secrets manager or HSM.
