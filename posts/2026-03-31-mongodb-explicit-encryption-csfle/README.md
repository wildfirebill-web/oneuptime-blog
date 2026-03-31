# How to Configure Explicit Encryption in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Encryption, CSFLE, Security, Database

Description: Learn how to use explicit client-side field level encryption in MongoDB to manually encrypt and decrypt specific field values using the ClientEncryption API.

---

Explicit encryption in MongoDB gives you direct control over which fields are encrypted and when. Unlike automatic encryption - where the driver handles everything based on a schema - explicit mode requires your application code to call encrypt and decrypt functions directly. This is useful when you need fine-grained control or when automatic encryption is not available.

## When to Use Explicit Encryption

- You need to encrypt different fields under different keys
- You want to encrypt values before storing them in arrays or nested paths
- You are using MongoDB Community Edition (automatic encryption requires Enterprise)
- You need to encrypt data before sending it to external systems

## Setup: Create a Data Encryption Key

```javascript
const { ClientEncryption } = require("mongodb-client-encryption")
const { MongoClient, Binary } = require("mongodb")

const client = new MongoClient("mongodb://localhost:27017")
await client.connect()

const keyVaultNamespace = "encryption.__keyVault"
const kmsProviders = {
  local: {
    key: Buffer.from(process.env.LOCAL_MASTER_KEY, "base64")
  }
}

const encryption = new ClientEncryption(client, { keyVaultNamespace, kmsProviders })

// Create or look up a DEK
const dataKeyId = await encryption.createDataKey("local", {
  keyAltNames: ["pii-key"]
})
```

## Encrypt a Value Explicitly

```javascript
// Encrypt the SSN field value before inserting
const encryptedSsn = await encryption.encrypt("123-45-6789", {
  algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic",
  keyAltName: "pii-key"
})

// Insert document with encrypted field
const db = client.db("myapp")
await db.collection("patients").insertOne({
  name: "Jane Doe",
  ssn: encryptedSsn,          // Stored as Binary ciphertext
  dob: new Date("1985-03-15") // Stored as plaintext
})
```

## Decrypt a Value Explicitly

When reading data, call `decrypt` on encrypted fields:

```javascript
const patient = await db.collection("patients").findOne({ name: "Jane Doe" })

// patient.ssn is Binary ciphertext - decrypt it
const plainSsn = await encryption.decrypt(patient.ssn)
console.log(plainSsn) // "123-45-6789"
```

## Query on Deterministically Encrypted Fields

Deterministic encryption produces the same ciphertext for the same plaintext, so equality queries work. Encrypt the query value before filtering:

```javascript
const encryptedQuery = await encryption.encrypt("123-45-6789", {
  algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic",
  keyAltName: "pii-key"
})

const result = await db.collection("patients").findOne({ ssn: encryptedQuery })
```

Random encryption cannot be queried - use it only for fields that do not need to be searched.

## Python Example

```python
from pymongo import MongoClient
from pymongo.encryption import ClientEncryption, Algorithm
import os, base64

client = MongoClient("mongodb://localhost:27017")

kms_providers = {
    "local": {"key": base64.b64decode(os.environ["LOCAL_MASTER_KEY"])}
}

encryption = ClientEncryption(
    kms_providers,
    "encryption.__keyVault",
    client,
    client.codec_options
)

# Encrypt
encrypted_ssn = encryption.encrypt(
    "123-45-6789",
    Algorithm.AEAD_AES_256_CBC_HMAC_SHA_512_Deterministic,
    key_alt_name="pii-key"
)

# Insert
client.myapp.patients.insert_one({
    "name": "Jane Doe",
    "ssn": encrypted_ssn
})

# Decrypt
doc = client.myapp.patients.find_one({"name": "Jane Doe"})
plain = encryption.decrypt(doc["ssn"])
print(plain)  # "123-45-6789"
```

## Explicit Encryption in Updates

Encrypt values before using them in update operations:

```javascript
const newEncryptedSsn = await encryption.encrypt("987-65-4321", {
  algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic",
  keyAltName: "pii-key"
})

await db.collection("patients").updateOne(
  { name: "Jane Doe" },
  { $set: { ssn: newEncryptedSsn } }
)
```

## Summary

Explicit CSFLE gives application-level control over field encryption and decryption using the `ClientEncryption` API. Encrypt values before inserting or updating, decrypt after reading, and use deterministic encryption for fields you need to query. This mode is available in MongoDB Community Edition and all drivers that support the encryption library.
