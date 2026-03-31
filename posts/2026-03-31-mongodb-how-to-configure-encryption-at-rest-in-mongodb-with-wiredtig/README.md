# How to Configure Encryption at Rest in MongoDB with WiredTiger

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Encryption, WiredTiger, Security, Enterprise

Description: Enable MongoDB Enterprise encrypted storage engine to protect data files on disk using AES-256 encryption with WiredTiger.

---

## About Encryption at Rest in MongoDB

MongoDB Enterprise supports native encryption at rest through the WiredTiger storage engine. Data files, indexes, journal, and temporary files are all encrypted transparently. This protects against physical disk theft or unauthorized storage access without changing application code.

Encryption at rest in MongoDB Enterprise uses AES-256-CBC or AES-256-GCM.

## Key Management Options

MongoDB supports two key management approaches:
1. **Local key management** - key stored in a file on the server (for testing only)
2. **KMIP** - keys stored in an external Key Management Interoperability Protocol server (HashiCorp Vault, Thales, AWS CloudHSM, etc.) - recommended for production

## Configuring Local Key Management (Testing Only)

```bash
# Generate a 32-byte base64 key
openssl rand -base64 32 > /etc/mongodb/local.key
chmod 600 /etc/mongodb/local.key
```

```yaml
# /etc/mongod.conf
storage:
  dbPath: /var/lib/mongodb
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      encryptionEnabled: true
      encryptionKeyFile: /etc/mongodb/local.key
```

## Configuring KMIP Key Management (Production)

```yaml
# /etc/mongod.conf
security:
  enableEncryption: true
  kmip:
    serverName: kmip.example.com
    port: 5696
    clientCertificateFile: /etc/ssl/kmip-client.pem
    serverCAFile: /etc/ssl/kmip-ca.crt
```

## Enabling Encryption on a New Deployment

Start with `--enableEncryption` flag or via config file from first startup. You cannot add encryption at rest to a running unencrypted deployment without a full resync.

```bash
mongod --enableEncryption \
  --kmipServerName kmip.example.com \
  --kmipPort 5696 \
  --kmipClientCertificateFile /etc/ssl/kmip-client.pem \
  --kmipServerCAFile /etc/ssl/kmip-ca.crt \
  --config /etc/mongod.conf
```

## Verifying Encryption Is Active

```javascript
db.adminCommand({ serverStatus: 1 }).encryptionAtRest
```

Output should show:

```javascript
{
  encryptionEnabled: true,
  encryptionKeyId: "...",
  keyManagementSystem: "kmip"
}
```

## Rotating Encryption Keys

Perform key rotation without downtime on MongoDB Enterprise:

```javascript
db.adminCommand({ rotateMasterKey: 1 });
```

This re-encrypts the database encryption key using a new master key from KMIP.

## Encrypted Backup Considerations

Backups taken with `mongodump` export decrypted BSON, so protect backup files with OS-level encryption or a separate mechanism. Hot backups via MongoDB Ops Manager or `mongodump --oplog` export plaintext - always encrypt backup storage.

## Summary

MongoDB Enterprise encryption at rest transparently encrypts all WiredTiger data files using AES-256. Enable it by setting `security.enableEncryption: true` and configuring a KMIP server for production key management. This must be enabled at first startup; it cannot be added to a running unencrypted deployment. Verify with `serverStatus().encryptionAtRest` and rotate keys periodically with `rotateMasterKey`.
