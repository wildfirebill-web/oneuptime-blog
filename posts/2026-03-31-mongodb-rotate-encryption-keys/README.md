# How to Rotate Encryption Keys in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Encryption, Key Rotation, CSFLE, Security

Description: Learn how to rotate Customer Master Keys and Data Encryption Keys in MongoDB CSFLE to maintain security compliance without losing access to encrypted data.

---

Key rotation is a security best practice that limits the blast radius if a key is compromised. In MongoDB CSFLE, there are two types of keys to rotate: the Customer Master Key (CMK) managed by your KMS, and Data Encryption Keys (DEKs) stored in the MongoDB key vault collection.

## Understanding the Two-Layer Key Hierarchy

```text
CMK (in KMS) -- wraps --> DEK (in MongoDB key vault) -- encrypts --> field data
```

- **Rotating the CMK**: Re-wraps existing DEKs with a new CMK. No field data is re-encrypted.
- **Rotating the DEK**: Creates a new DEK and re-encrypts all affected field data using the new key.

## Rotating the CMK (KMS-level Rotation)

### AWS KMS - Automatic Rotation

```bash
# Enable annual automatic rotation
aws kms enable-key-rotation --key-id YOUR-KEY-ID

# Check rotation status
aws kms get-key-rotation-status --key-id YOUR-KEY-ID
```

AWS KMS transparently decrypts DEKs wrapped with old CMK versions and re-encrypts with the current version when needed.

### Azure Key Vault - Rotation Policy

```bash
az keyvault key rotation-policy update \
  --vault-name my-mongo-kv \
  --name mongo-csfle-key \
  --value '{"lifetimeActions":[{"trigger":{"timeAfterCreate":"P1Y"},"action":{"type":"Rotate"}}]}'
```

### GCP KMS - Rotation Schedule

```bash
gcloud kms keys update mongo-csfle-key \
  --location global \
  --keyring mongo-csfle-ring \
  --rotation-period 365d \
  --next-rotation-time $(date -d "+1 year" --iso-8601)
```

## Rewrapping DEKs After CMK Rotation

MongoDB 6.0+ provides `rewrapManyDataKey` to re-wrap DEKs with a new CMK without re-encrypting field data:

```javascript
const { ClientEncryption } = require("mongodb-client-encryption")
const { MongoClient } = require("mongodb")

const client = new MongoClient("mongodb://localhost:27017")
await client.connect()

const kmsProviders = {
  aws: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
  }
}

const encryption = new ClientEncryption(client, {
  keyVaultNamespace: "encryption.__keyVault",
  kmsProviders
})

// Rewrap all DEKs currently using old AWS key with a new CMK
const result = await encryption.rewrapManyDataKey(
  {},  // Filter: empty = all DEKs
  {
    provider: "aws",
    masterKey: {
      region: "us-east-1",
      key: "arn:aws:kms:us-east-1:123456789012:key/NEW-KEY-ID"
    }
  }
)

console.log(`Rewrapped ${result.bulkWriteResult.modifiedCount} DEKs`)
```

## Rotating a DEK (Re-encrypting Field Data)

When you need to rotate the DEK itself, you must re-encrypt all documents that use the old DEK. This requires a background migration:

```javascript
// 1. Create new DEK
const newDekId = await encryption.createDataKey("aws", {
  masterKey: { region: "us-east-1", key: "arn:aws:kms:..." },
  keyAltNames: ["user-ssn-key-v2"]
})

// 2. Re-encrypt field values in bulk (simplified example)
const patients = client.db("myapp").collection("patients")
const cursor = patients.find({})

for await (const doc of cursor) {
  if (!doc.ssn) continue

  // Decrypt with old key
  const plaintext = await encryption.decrypt(doc.ssn)

  // Re-encrypt with new DEK
  const reEncrypted = await encryption.encrypt(plaintext, {
    algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic",
    keyAltName: "user-ssn-key-v2"
  })

  await patients.updateOne({ _id: doc._id }, { $set: { ssn: reEncrypted } })
}

// 3. Delete old DEK after confirming all data is migrated
const keyVault = client.db("encryption").collection("__keyVault")
await keyVault.deleteOne({ keyAltNames: "user-ssn-key" })
```

## Verify After Rotation

After rewrapping or re-encrypting, test that data is still accessible:

```javascript
const patient = await encryptedClient.db("myapp").collection("patients")
                 .findOne({ name: "Test Patient" })
console.log(patient.ssn)  // Should return decrypted value
```

## Summary

Key rotation in MongoDB CSFLE involves two layers: CMK rotation at the KMS level (AWS, Azure, GCP) and DEK rotation for the MongoDB key vault. Use `rewrapManyDataKey` to efficiently re-wrap DEKs with a new CMK. For full DEK rotation, run a background job to re-encrypt affected documents with new keys. Test decryption after every rotation step.
