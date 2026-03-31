# How to Define Encrypted Field Maps for Queryable Encryption in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Queryable Encryption, Encryption, Security, Schema

Description: Learn how to define and structure encrypted field maps for MongoDB Queryable Encryption, including field-level configuration options and best practices.

---

An encrypted field map (EFM) is the schema that tells the MongoDB driver which fields to encrypt, what type they are, and what query operations they support. Defining it correctly is the foundation of a working Queryable Encryption setup.

## Basic Structure

An encrypted field map is a JavaScript object keyed by `"<database>.<collection>"`:

```javascript
const encryptedFieldsMap = {
  "hr.employees": {
    fields: [
      {
        path: "ssn",
        bsonType: "string",
        queries: [{ queryType: "equality" }]
      },
      {
        path: "salary",
        bsonType: "int",
        queries: [
          {
            queryType: "range",
            sparsity: 1,
            min: 20000,
            max: 1000000,
            trimFactor: 6
          }
        ]
      },
      {
        path: "medicalNotes",
        bsonType: "string"
        // No queries: field is encrypted but not queryable
      }
    ]
  }
};
```

## Field Options Explained

| Option        | Required | Description                                             |
|---------------|----------|---------------------------------------------------------|
| `path`        | Yes      | Dot-notation path to the field                          |
| `bsonType`    | Yes      | BSON type of the plaintext value                        |
| `queries`     | No       | Array of query configurations (omit for unqueryable)   |
| `keyId`       | No       | UUID of the DEK to use (overrides collection-level DEK) |

## Nested Field Paths

You can encrypt nested document fields using dot notation:

```javascript
{
  path: "address.zipCode",
  bsonType: "string",
  queries: [{ queryType: "equality" }]
}
```

You cannot encrypt a top-level field and a nested sub-field of that same field. The path must point to a leaf value.

## Providing a keyId Per Field

By default, all fields in a collection can share a single Data Encryption Key (DEK). For tighter key management, assign a specific DEK per field:

```javascript
const ssnKeyId = await encryption.createDataKey("aws", { ... });
const salaryKeyId = await encryption.createDataKey("aws", { ... });

const encryptedFieldsMap = {
  "hr.employees": {
    fields: [
      {
        path: "ssn",
        bsonType: "string",
        keyId: ssnKeyId,
        queries: [{ queryType: "equality" }]
      },
      {
        path: "salary",
        bsonType: "int",
        keyId: salaryKeyId,
        queries: [{ queryType: "range", min: 0, max: 1000000, sparsity: 1 }]
      }
    ]
  }
};
```

Using separate DEKs allows you to rotate keys per field and limit blast radius if a key is compromised.

## Creating the Collection with the EFM

Pass the field map to `createCollection` so MongoDB creates the required metadata collections:

```javascript
await db.createCollection("employees", {
  encryptedFields: encryptedFieldsMap["hr.employees"]
});
```

Do not create the collection with `insertOne` first - the metadata collections must exist before any QE writes.

## Attaching the EFM to the Client

Pass the same map in `autoEncryption` when constructing the `MongoClient`:

```javascript
const client = new MongoClient(uri, {
  autoEncryption: {
    keyVaultNamespace: "encryption.__keyVault",
    kmsProviders: { local: { key: masterKey } },
    encryptedFieldsMap
  }
});
```

## Summary

The encrypted field map is the central configuration artifact for MongoDB Queryable Encryption. Define each sensitive field with its BSON type, optional query type, and optionally a dedicated DEK. Use dot notation for nested fields, and always create the collection through the driver to ensure metadata collections are set up correctly. Assigning per-field DEKs improves key rotation flexibility and limits exposure if a single key is compromised.
