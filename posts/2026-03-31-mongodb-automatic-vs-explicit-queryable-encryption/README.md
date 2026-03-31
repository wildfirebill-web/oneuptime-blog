# How to Configure Automatic vs Explicit Encryption in MongoDB Queryable Encryption

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Queryable Encryption, Security, Encryption, Node.js

Description: Understand the difference between automatic and explicit Queryable Encryption in MongoDB, and learn when to use each approach with practical Node.js examples.

---

MongoDB Queryable Encryption (QE) allows you to encrypt sensitive fields while still being able to run equality and range queries on them. There are two modes: automatic and explicit. Choosing the right one depends on your driver support and operational requirements.

## Automatic Queryable Encryption

With automatic QE, the MongoDB driver handles encryption and decryption transparently. You define an `encryptedFieldsMap` when creating the client, and the driver encrypts matching fields before they hit the wire.

```javascript
const { MongoClient } = require("mongodb");
const { ClientEncryption } = require("mongodb-client-encryption");

const encryptedFieldsMap = {
  "mydb.patients": {
    fields: [
      {
        path: "ssn",
        bsonType: "string",
        queries: [{ queryType: "equality" }]
      }
    ]
  }
};

const client = new MongoClient(uri, {
  autoEncryption: {
    keyVaultNamespace: "encryption.__keyVault",
    kmsProviders: { local: { key: localMasterKey } },
    encryptedFieldsMap
  }
});

// Writes: ssn is encrypted automatically
await client.db("mydb").collection("patients").insertOne({
  name: "Alice",
  ssn: "123-45-6789"
});

// Reads: driver decrypts ssn automatically
const result = await client.db("mydb").collection("patients").findOne({
  ssn: "123-45-6789"
});
console.log(result.ssn); // "123-45-6789"
```

## Explicit Queryable Encryption

With explicit QE, you call encryption and decryption methods manually using the `ClientEncryption` API. This gives you fine-grained control but requires more application code.

```javascript
const encryption = new ClientEncryption(client, {
  keyVaultNamespace: "encryption.__keyVault",
  kmsProviders: { local: { key: localMasterKey } }
});

// Encrypt manually before insert
const encryptedSSN = await encryption.encrypt("123-45-6789", {
  algorithm: "Indexed",
  keyId: dataKeyId,
  contentionFactor: 8
});

await regularClient.db("mydb").collection("patients").insertOne({
  name: "Alice",
  ssn: encryptedSSN
});

// Decrypt after retrieval
const doc = await regularClient.db("mydb").collection("patients").findOne({ name: "Alice" });
const plainSSN = await encryption.decrypt(doc.ssn);
```

## Key Differences

| Feature               | Automatic                    | Explicit                     |
|-----------------------|------------------------------|------------------------------|
| Driver support needed | Full (4.x+)                  | Any driver with ClientEncryption |
| Code complexity       | Low                          | High                         |
| Schema enforcement    | Yes - via encryptedFieldsMap | No - developer responsibility |
| Mixed plaintext risk  | Low                          | Higher                       |
| Query support         | Full (equality, range)       | Full                         |

## When to Use Each

Use **automatic** encryption when:
- You use a supported driver (Node.js, Java, Python, Go, C#)
- You want the driver to enforce encryption rules consistently
- You prefer less application code

Use **explicit** encryption when:
- Your driver or environment does not fully support automatic QE
- You need to encrypt fields at different times or conditionally
- You are building a custom encryption workflow

## Setting Up the Key Vault

Both modes require a key vault collection and a data encryption key (DEK):

```javascript
const encryption = new ClientEncryption(client, {
  keyVaultNamespace: "encryption.__keyVault",
  kmsProviders: { local: { key: localMasterKey } }
});

const dataKeyId = await encryption.createDataKey("local", {
  keyAltNames: ["patient-ssn-key"]
});
```

## Summary

Automatic Queryable Encryption is easier to implement and less error-prone because the driver enforces the encryption schema on every read and write. Explicit encryption is more flexible but puts the burden of consistent encryption on the developer. For most production use cases, automatic encryption is the recommended approach when using a supported driver.
