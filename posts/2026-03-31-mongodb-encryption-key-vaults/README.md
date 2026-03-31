# How to Manage Encryption Key Vaults in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Encryption, Key Vault, CSFLE, Security

Description: Learn how MongoDB encryption key vaults work, how to create and manage Data Encryption Keys, and how to set up access controls on the key vault collection.

---

The key vault is a MongoDB collection that stores Data Encryption Keys (DEKs) used for Client-Side Field Level Encryption. DEKs are themselves encrypted by a Customer Master Key (CMK) from a KMS provider, so the MongoDB server never has access to plaintext keys.

## Key Vault Architecture

```text
Application <-> MongoDB Driver (with CSFLE)
                    |
                    v
           Key Vault Collection (encryption.__keyVault)
                    |  (DEKs stored as encrypted BinData)
                    v
           KMS Provider (AWS KMS / Azure Key Vault / GCP KMS / local)
                    |  (CMK wraps/unwraps DEKs)
```

## Create the Key Vault Collection with an Index

The key vault collection must have a unique index on `keyAltNames`:

```javascript
const { MongoClient } = require("mongodb")
const client = new MongoClient("mongodb://localhost:27017")
await client.connect()

const keyVaultDb = client.db("encryption")
await keyVaultDb.createCollection("__keyVault")
await keyVaultDb.collection("__keyVault").createIndex(
  { keyAltNames: 1 },
  { unique: true, partialFilterExpression: { keyAltNames: { $exists: true } } }
)
```

## Create Data Encryption Keys

```javascript
const { ClientEncryption } = require("mongodb-client-encryption")

const kmsProviders = {
  local: {
    key: Buffer.from(process.env.LOCAL_MASTER_KEY, "base64")
  }
}

const encryption = new ClientEncryption(client, {
  keyVaultNamespace: "encryption.__keyVault",
  kmsProviders
})

// Create a DEK with an alternate name for easy reference
const dek1 = await encryption.createDataKey("local", {
  keyAltNames: ["user-pii-key"]
})

const dek2 = await encryption.createDataKey("local", {
  keyAltNames: ["payment-key"]
})

console.log("user-pii-key DEK ID:", dek1)
console.log("payment-key DEK ID:", dek2)
```

## List DEKs in the Key Vault

```javascript
const keyVault = client.db("encryption").collection("__keyVault")
const keys = await keyVault.find({}).toArray()

keys.forEach(k => {
  console.log("Key ID:", k._id, "Alt names:", k.keyAltNames, "Created:", k.creationDate)
})
```

## Look Up a DEK by Alt Name

```javascript
const key = await keyVault.findOne({ keyAltNames: "user-pii-key" })
console.log("DEK document:", key)
```

## Restrict Key Vault Access

The key vault should only be readable by authorized encryption users. Create a dedicated user:

```javascript
use admin
db.createUser({
  user: "encryptionAdmin",
  pwd: "VaultAdminPass!",
  roles: [
    { role: "readWrite", db: "encryption" }
  ]
})
```

Application users should only have `read` on the key vault, not `readWrite`. Only the encryption admin creates new DEKs.

```javascript
use admin
db.createUser({
  user: "appservice",
  pwd: "AppPass!",
  roles: [
    { role: "readWrite", db: "myapp" },
    { role: "read", db: "encryption" }  // Read DEKs but cannot create new ones
  ]
})
```

## Delete a DEK

Deleting a DEK makes all data encrypted with it permanently unreadable. Do this only intentionally for data deletion compliance:

```javascript
await keyVault.deleteOne({ keyAltNames: "old-payment-key" })
```

## Rotate a DEK

MongoDB does not have built-in automatic DEK rotation, but you can re-wrap a DEK with a new CMK if using AWS KMS, Azure, or GCP. For manual rotation, create a new DEK and re-encrypt the affected fields using a background migration job.

## Back Up the Key Vault

The key vault is a regular MongoDB collection - back it up with `mongodump`:

```bash
mongodump \
  --host localhost:27017 \
  --db encryption \
  --collection __keyVault \
  --out /backup/keyvault-$(date +%F)
```

Store key vault backups in a secure, separate location from the encrypted data.

## Summary

The MongoDB key vault stores DEKs encrypted by an external KMS provider. Create it with a unique index on `keyAltNames`, restrict write access to authorized administrators, and back it up separately from your application data. Losing the key vault or the CMK without a backup makes encrypted data permanently unrecoverable.
