# How to Use ClickHouse Transparent Data Encryption

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Security, Encryption, Database, Infrastructure

Description: Learn how to enable ClickHouse Transparent Data Encryption to protect data at rest using AES-128 or AES-256, including key management and disk configuration.

## Introduction

Transparent Data Encryption (TDE) encrypts data files on disk so that anyone with physical access to the storage media cannot read the raw data. In ClickHouse, TDE is implemented at the virtual disk layer. You define an encrypted disk in the server configuration, point your tables at that disk through a storage policy, and ClickHouse handles encryption and decryption transparently during reads and writes. Applications and queries require no changes.

ClickHouse supports AES-128-CTR and AES-256-CTR for data encryption. Keys can be supplied as inline hex values for simple setups or loaded from a key management service (KMS) in production.

## Prerequisites

- ClickHouse 21.4 or newer
- Root or `clickhouse` OS user access to edit `/etc/clickhouse-server/config.xml` or files under `/etc/clickhouse-server/config.d/`
- Sufficient disk space (encrypted disks have negligible overhead in terms of space)

## Step 1 - Define an Encrypted Disk

Add a storage configuration file that declares a local disk and an encrypted overlay on top of it.

```xml
<!-- /etc/clickhouse-server/config.d/encrypted_disk.xml -->
<clickhouse>
    <storage_configuration>
        <disks>
            <!-- Base local disk (plain, unencrypted) -->
            <local_plain>
                <type>local</type>
                <path>/var/lib/clickhouse/store/plain/</path>
            </local_plain>

            <!-- Encrypted overlay on top of the local disk -->
            <encrypted_local>
                <type>encrypted</type>
                <disk>local_plain</disk>
                <path>encrypted/</path>
                <algorithm>AES_128_CTR</algorithm>
                <!-- 32-character hex key for AES-128 (16 bytes = 32 hex chars) -->
                <key_hex>0123456789abcdef0123456789abcdef</key_hex>
            </encrypted_local>
        </disks>

        <policies>
            <encrypted_policy>
                <volumes>
                    <main>
                        <disk>encrypted_local</disk>
                    </main>
                </volumes>
            </encrypted_policy>
        </policies>
    </storage_configuration>
</clickhouse>
```

For AES-256-CTR, use a 64-character hex key (32 bytes):

```xml
<algorithm>AES_256_CTR</algorithm>
<key_hex>0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef</key_hex>
```

## Step 2 - Restart ClickHouse

```bash
sudo systemctl restart clickhouse-server
sudo systemctl status clickhouse-server
```

Confirm no errors appear in the log:

```bash
sudo journalctl -u clickhouse-server -n 50
```

## Step 3 - Create a Table on the Encrypted Disk

Use the `SETTINGS storage_policy` clause to direct the table to the encrypted storage policy.

```sql
CREATE TABLE sensitive_events
(
    event_id    UUID DEFAULT generateUUIDv4(),
    user_id     String,
    email       String,
    action      String,
    payload     String,
    created_at  DateTime DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id)
SETTINGS storage_policy = 'encrypted_policy';
```

All data parts written to this table are encrypted on disk automatically.

## Step 4 - Verify Encryption is Active

Check that the disk type is recognized correctly:

```sql
SELECT
    name,
    type,
    path,
    free_space,
    total_space
FROM system.disks
WHERE name IN ('local_plain', 'encrypted_local');
```

Confirm the table uses the encrypted policy:

```sql
SELECT
    name,
    storage_policy
FROM system.tables
WHERE name = 'sensitive_events';
```

Inspect a data part file directly on the OS to confirm it is not readable:

```bash
# This should output unreadable binary, not SQL or column data
sudo xxd /var/lib/clickhouse/store/plain/encrypted/data/default/sensitive_events/all_1_1_0/email.bin | head -5
```

## Key Rotation

Rotating the encryption key requires decrypting and re-encrypting data parts. ClickHouse supports key rotation via multiple key IDs in the configuration.

```xml
<encrypted_local>
    <type>encrypted</type>
    <disk>local_plain</disk>
    <path>encrypted/</path>
    <algorithm>AES_256_CTR</algorithm>
    <!-- Old key - keep until all parts are re-encrypted -->
    <key_hex id="0">0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef</key_hex>
    <!-- New key - will be used for all new writes -->
    <key_hex id="1" current="true">fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210</key_hex>
</encrypted_local>
```

After restarting ClickHouse with the new configuration, existing parts remain readable using the old key ID (stored in part metadata). New parts are written with the new key. Re-encrypt old parts by running a merge:

```sql
OPTIMIZE TABLE sensitive_events FINAL;
```

Once all parts have been merged and rewritten with the new key, remove the old key from the configuration.

## Using an External Key Management Service

For production deployments, storing keys in the configuration file is not ideal. ClickHouse supports loading keys from environment variables or external files, which can be managed by a secrets manager such as HashiCorp Vault or AWS Secrets Manager.

```xml
<encrypted_local>
    <type>encrypted</type>
    <disk>local_plain</disk>
    <path>encrypted/</path>
    <algorithm>AES_256_CTR</algorithm>
    <!-- Load key from environment variable set by your secrets manager -->
    <key_hex from_env="CLICKHOUSE_DISK_ENCRYPTION_KEY"/>
</encrypted_local>
```

Inject the environment variable into the ClickHouse service unit:

```bash
# /etc/systemd/system/clickhouse-server.service.d/override.conf
[Service]
EnvironmentFile=/etc/clickhouse-server/secrets.env
```

```bash
# /etc/clickhouse-server/secrets.env (mode 600, owned by clickhouse)
CLICKHOUSE_DISK_ENCRYPTION_KEY=fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210
```

## Encrypting S3-Backed Disks

If you use S3 or S3-compatible storage, the same encrypted disk layer applies.

```xml
<disks>
    <s3_plain>
        <type>s3</type>
        <endpoint>https://s3.amazonaws.com/my-bucket/clickhouse/</endpoint>
        <access_key_id>AKIAIOSFODNN7EXAMPLE</access_key_id>
        <secret_access_key>wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY</secret_access_key>
    </s3_plain>

    <s3_encrypted>
        <type>encrypted</type>
        <disk>s3_plain</disk>
        <path>encrypted/</path>
        <algorithm>AES_256_CTR</algorithm>
        <key_hex from_env="CLICKHOUSE_S3_ENCRYPTION_KEY"/>
    </s3_encrypted>
</disks>
```

This adds a client-side encryption layer on top of any S3-side encryption you may also have enabled, giving you defense in depth.

## Performance Impact

Encryption in CTR mode is parallelizable and hardware-accelerated on modern CPUs via AES-NI instructions. In practice, the throughput penalty is under 5% for sequential reads and writes. You can confirm AES-NI is available on your host:

```bash
grep -m1 aes /proc/cpuinfo
```

If `aes` appears in the flags line, hardware acceleration is active and TDE overhead is minimal.

## Summary

ClickHouse Transparent Data Encryption protects data at rest by wrapping any disk type - local or S3 - with an AES-CTR encrypted overlay defined in the storage configuration. Tables are assigned to the encrypted storage policy and require no application changes. For production use, load encryption keys from environment variables managed by a secrets manager and rotate keys periodically using ClickHouse's multi-key configuration support.
