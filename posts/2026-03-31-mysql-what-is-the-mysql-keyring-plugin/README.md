# What Is the MySQL Keyring Plugin

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Keyring Plugin, Security, Encryption Key Management

Description: The MySQL Keyring Plugin provides a secure key management interface that stores and retrieves encryption keys used by MySQL's Transparent Data Encryption and other features.

---

## Overview

The MySQL Keyring Plugin (and the newer Keyring Component framework in MySQL 8.0.24+) provides a secure storage backend for encryption keys used by MySQL features like Transparent Data Encryption (TDE), binary log encryption, and undo log encryption.

The keyring acts as a vault: MySQL requests keys by ID, and the keyring plugin retrieves them from wherever it stores them - a local file, an encrypted file, or an external key management system.

## Keyring Plugin Options

MySQL supports several keyring backends:

| Plugin/Component | Storage |
|------------------|---------|
| `keyring_file` | Plain file on local disk |
| `keyring_encrypted_file` | AES-encrypted file on local disk |
| `keyring_okv` | Oracle Key Vault (OKV) |
| `keyring_aws` | AWS Key Management Service |
| `keyring_hashicorp` | HashiCorp Vault |
| `component_keyring_file` | Component version of file keyring |
| `component_keyring_encrypted_file` | Component version of encrypted file keyring |

## Installing keyring_file (Development Use)

```text
[mysqld]
early-plugin-load = keyring_file.so
keyring_file_data = /var/lib/mysql-keyring/keyring
```

The `early-plugin-load` ensures the keyring is available before InnoDB initializes.

## Installing keyring_encrypted_file

```text
[mysqld]
early-plugin-load = keyring_encrypted_file.so
keyring_encrypted_file_data = /var/lib/mysql-keyring/keyring.enc
keyring_encrypted_file_password = your_secure_password
```

## Using the Component Keyring (MySQL 8.0.24+)

```sql
-- Install the component
INSTALL COMPONENT 'file://component_keyring_file';

-- Create configuration file
-- /var/lib/mysql-keyring/component_keyring_file.cnf
```

Configuration file (`component_keyring_file.cnf`):

```json
{
  "path": "/var/lib/mysql-keyring/keyring",
  "read_only": false
}
```

## AWS KMS Keyring Setup

```text
[mysqld]
early-plugin-load = keyring_aws.so
keyring_aws_cmk_id = arn:aws:kms:us-east-1:123456789:key/abc-def
keyring_aws_region = us-east-1
```

```bash
# AWS credentials must be accessible
export AWS_ACCESS_KEY_ID=your_key
export AWS_SECRET_ACCESS_KEY=your_secret
```

## HashiCorp Vault Keyring Setup

```text
[mysqld]
early-plugin-load = keyring_hashicorp.so
keyring_hashicorp_server_url = https://vault.example.com:8200
keyring_hashicorp_auth_path = /v1/auth/cert/login
keyring_hashicorp_store_path = /v1/secret/mysql
keyring_hashicorp_ca_path = /etc/ssl/certs/vault-ca.pem
```

## Verifying the Keyring Is Active

```sql
-- Check loaded keyring plugins
SELECT plugin_name, plugin_status
FROM information_schema.plugins
WHERE plugin_name LIKE 'keyring%';

-- Test key generation
SELECT keyring_key_generate('MyTestKey', 'AES', 32);
SELECT keyring_key_remove('MyTestKey');
```

## Keyring Functions

MySQL provides keyring UDFs for manual key operations:

```sql
-- Generate a key
SELECT keyring_key_generate('app_encryption_key', 'AES', 32);

-- Fetch a key (returns NULL if not found)
SELECT keyring_key_fetch('app_encryption_key');

-- Remove a key
SELECT keyring_key_remove('app_encryption_key');
```

## Rotating the Master Key

```sql
-- Rotate InnoDB master key (uses currently active keyring)
ALTER INSTANCE ROTATE INNODB MASTER KEY;
```

## Backing Up the Keyring File

The keyring file must be backed up alongside MySQL data. If you lose the keyring file, encrypted data is permanently inaccessible:

```bash
# Always back up together
cp /var/lib/mysql-keyring/keyring /backup/keyring-$(date +%Y%m%d)
```

## Summary

The MySQL Keyring Plugin is the foundation of MySQL's encryption key management infrastructure. It abstracts key storage behind a unified interface that supports file-based, cloud-based (AWS KMS), and enterprise key vaults (HashiCorp, Oracle). The keyring must be configured with `early-plugin-load` so it is available before InnoDB starts and must be backed up with the same care as the data it protects.
