# How to Implement Right to Be Forgotten (Data Deletion) in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, GDPR, Data Deletion

Description: Implement GDPR's right to erasure in MongoDB with transactional deletion workflows, cascade patterns, backup scrubbing, and audit trail maintenance.

---

## What Right to Be Forgotten Requires

GDPR Article 17 grants individuals the right to request deletion of their personal data. For MongoDB systems, this means deleting or anonymizing data across all collections that contain that person's information, including backup systems within a reasonable timeframe. You must also log the deletion request while not retaining the data itself.

## Identifying All Personal Data Locations

Before implementing deletion, catalog where personal data lives:

```text
users             - direct personal data (name, email, phone, address)
orders            - customer ID references + shipping address
audit_logs        - IP addresses, user IDs
sessions          - user ID, IP, user agent
consent_records   - user ID, IP, timestamp
notifications     - email address, user ID
activity_events   - user ID, action timestamps
```

## Deletion Request Intake

Store deletion requests separately before processing:

```javascript
async function submitDeletionRequest(userId, requestedBy = 'user') {
  const request = await db.collection('deletion_requests').insertOne({
    userId: new ObjectId(userId),
    requestedAt: new Date(),
    requestedBy,
    status: 'pending',
    completedAt: null,
  })
  return request.insertedId
}
```

## Transactional Deletion Across Collections

```javascript
async function processErasureRequest(requestId) {
  const request = await db.collection('deletion_requests').findOne({
    _id: new ObjectId(requestId),
    status: 'pending',
  })
  if (!request) throw new Error('Request not found or already processed')

  const userId = request.userId
  const session = client.startSession()
  session.startTransaction()

  try {
    // 1. Delete the user account
    await db.collection('users').deleteOne({ _id: userId }, { session })

    // 2. Anonymize orders (keep for financial records, remove PII)
    await db.collection('orders').updateMany(
      { userId },
      {
        $unset: {
          shippingAddress: "",
          billingAddress: "",
          customerEmail: "",
        },
        $set: { userId: "DELETED_" + Date.now() },
      },
      { session }
    )

    // 3. Delete sessions
    await db.collection('sessions').deleteMany({ userId }, { session })

    // 4. Delete consent records (the right to erasure supersedes consent records)
    await db.collection('consent_records').deleteMany({ userId }, { session })

    // 5. Anonymize activity events
    await db.collection('activity_events').updateMany(
      { userId },
      { $unset: { userId: "", ipAddress: "" }, $set: { anonymized: true } },
      { session }
    )

    // 6. Mark the deletion request as complete (no personal data retained)
    await db.collection('deletion_requests').updateOne(
      { _id: new ObjectId(requestId) },
      { $set: { status: 'completed', completedAt: new Date() } },
      { session }
    )

    await session.commitTransaction()
    console.log(`Erasure complete for request ${requestId}`)
  } catch (err) {
    await session.abortTransaction()
    await db.collection('deletion_requests').updateOne(
      { _id: new ObjectId(requestId) },
      { $set: { status: 'failed', error: err.message } }
    )
    throw err
  } finally {
    session.endSession()
  }
}
```

## Handling Backup and Archive Systems

For backups, document your retention and scrubbing policy:

```bash
#!/bin/bash
# backup_scrub.sh - Run after erasure to remove data from recent backups
# Only needed for backups within retention window

BACKUP_DIR="/backup/mongodb"
USER_ID="$1"

for backup in $(ls -t "$BACKUP_DIR" | head -7); do
  echo "Scrubbing $backup..."
  mongorestore --uri "$SCRUB_MONGO_URI" --drop "$BACKUP_DIR/$backup"
  mongosh "$SCRUB_MONGO_URI" --eval "db.users.deleteOne({_id: ObjectId('$USER_ID')})"
  mongodump --uri "$SCRUB_MONGO_URI" --out "$BACKUP_DIR/${backup}_scrubbed"
  rm -rf "$BACKUP_DIR/$backup"
  mv "$BACKUP_DIR/${backup}_scrubbed" "$BACKUP_DIR/$backup"
done
```

## Sending Confirmation

```javascript
async function sendErasureConfirmation(email, requestId) {
  // Send before purging the email address
  await emailService.send({
    to: email,
    subject: 'Your data deletion request has been completed',
    body: `Your personal data has been deleted. Reference: ${requestId}`,
  })
}
```

## Summary

GDPR right-to-erasure in MongoDB requires a coordinated, transactional approach across all collections holding personal data. Use a deletion_requests collection for intake and audit trail, execute the actual deletion in a MongoDB transaction to ensure atomicity, distinguish between deletable data (sessions, consents, user accounts) and data that must be retained for legal reasons (financial records that get anonymized instead), and address backup scrubbing within your documented retention window.
