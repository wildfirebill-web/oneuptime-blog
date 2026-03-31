# How to Use Encrypted Disk in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Security, Storage, Encryption, Database, Configuration

Description: Learn how to configure encrypted disks in ClickHouse to protect data at rest using AES-128 or AES-256 encryption without changing your query logic.

---

ClickHouse supports transparent data-at-rest encryption through a virtual disk type called `encrypted`. An encrypted disk wraps any existing disk (local, S3, HDFS) and encrypts all data written through it using AES-CTR mode. Queries and inserts work identically - encryption and decryption happen transparently in the storage layer.

## How Encrypted Disk Works

The encrypted disk is a wrapper. It sits on top of a real disk (called the underlying disk) and intercepts all read and write operations. Data is encrypted before being written to the underlying disk and decrypted when read back. ClickHouse stores encrypted data and a small metadata header per file. The encryption key never touches the data files.

Supported algorithms:

| Algorithm | Key length |
|-----------|-----------|
| AES_128_CTR | 16 bytes |
| AES_192_CTR | 24 bytes |
| AES_256_CTR | 32 bytes |

## Generating an Encryption Key

Generate a 32-byte (256-bit) key and hex-encode it:

```bash
openssl rand -hex 32
```

Example output (use your own generated value):

```text
a3f2c1d4e5b6a7f8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
```

## Configuring the Encrypted Disk

Add the encrypted disk to `/etc/clickhouse-server/config.d/encrypted_disk.xml`:

```xml
<clickhouse>
  <storage_configuration>
    <disks>
      <!-- The underlying real disk -->
      <local>
        <path>/var/lib/clickhouse/</path>
      </local>

      <!-- Encrypted wrapper around the local disk -->
      <encrypted_local>
        <type>encrypted</type>
        <disk>local</disk>
        <!-- Path on the underlying disk where encrypted data is stored -->
        <path>encrypted/</path>
        <!-- AES-256-CTR encryption -->
        <algorithm>AES_256_CTR</algorithm>
        <!-- Hex-encoded 32-byte key -->
        <key_hex>a3f2c1d4e5b6a7f8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2</key_hex>
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

## Using Multiple Encryption Keys

ClickHouse supports key rotation. Define multiple keys and mark one as current:

```xml
<encrypted_local>
  <type>encrypted</type>
  <disk>local</disk>
  <path>encrypted/</path>
  <algorithm>AES_256_CTR</algorithm>
  <!-- New current key (id=2) -->
  <key_hex id="2">b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5</key_hex>
  <!-- Old key (id=1) still needed to read older data -->
  <key_hex id="1">a3f2c1d4e5b6a7f8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2</key_hex>
  <!-- Tell ClickHouse which key to use for new writes -->
  <current_key_id>2</current_key_id>
</encrypted_local>
```

During key rotation, old data encrypted with key id=1 can still be read. New data is written with key id=2. To fully rotate, rewrite old data (using `OPTIMIZE TABLE FINAL`) and then remove key id=1 from config.

## Creating a Table on the Encrypted Disk

```sql
CREATE TABLE sensitive_data
(
    record_id  UInt64,
    user_id    UInt64,
    ssn        String,
    dob        Date,
    created_at DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, record_id)
SETTINGS storage_policy = 'encrypted_policy';
```

Inserts and queries work identically to any other table. Encryption is invisible to SQL.

## Encrypting an S3 Disk

Wrap an S3 disk with encryption for encrypted object storage:

```xml
<disks>
  <s3_base>
    <type>s3</type>
    <endpoint>https://s3.us-east-1.amazonaws.com/my-bucket/data/</endpoint>
    <access_key_id from_env="AWS_ACCESS_KEY_ID"/>
    <secret_access_key from_env="AWS_SECRET_ACCESS_KEY"/>
    <metadata_path>/var/lib/clickhouse/disks/s3_base/</metadata_path>
  </s3_base>

  <s3_encrypted>
    <type>encrypted</type>
    <disk>s3_base</disk>
    <path>encrypted/</path>
    <algorithm>AES_256_CTR</algorithm>
    <key_hex>a3f2c1d4e5b6a7f8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2</key_hex>
  </s3_encrypted>
</disks>
```

## Verifying Encrypted Disk Is Active

```sql
SELECT name, type, path
FROM system.disks
WHERE name = 'encrypted_local';
```

Confirm data is on the encrypted disk:

```sql
SELECT
    name         AS part,
    disk_name,
    formatReadableSize(bytes_on_disk) AS size
FROM system.parts
WHERE active = 1
  AND table = 'sensitive_data'
  AND database = currentDatabase();
```

## Storing the Key Securely

Embedding the raw key in XML is convenient but not ideal for production. Prefer reading the key from an environment variable:

```xml
<key_hex from_env="CLICKHOUSE_DISK_ENCRYPTION_KEY"/>
```

Set the variable in the systemd service override:

```bash
systemctl edit clickhouse-server
```

```text
[Service]
Environment="CLICKHOUSE_DISK_ENCRYPTION_KEY=a3f2c1d4e5b6a7f8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
```

Alternatively, use a secrets management tool (HashiCorp Vault, AWS Secrets Manager) to inject the key at deploy time.

## Important Notes

- Encryption adds a small CPU overhead per read/write operation, typically under 5% on modern hardware with AES-NI support.
- Losing the encryption key means losing access to all data on the encrypted disk permanently. Back up the key securely and separately from the data.
- The `encrypted` disk type does not encrypt metadata stored in ZooKeeper (for ReplicatedMergeTree tables).
- Backup tools must support encrypted disks. When using `BACKUP ... TO` with an encrypted disk, the backup is stored encrypted if the destination is also encrypted.

## Summary

ClickHouse's encrypted disk is a transparent wrapper that applies AES-CTR encryption to any underlying disk, including local disks and S3. Configure it in `storage_configuration`, apply it to a storage policy, and assign that policy to tables holding sensitive data. Use environment variables to store encryption keys out of config files, and plan a key rotation strategy before deploying to production.
