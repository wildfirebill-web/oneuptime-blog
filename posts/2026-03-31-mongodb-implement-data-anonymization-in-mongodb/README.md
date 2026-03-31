# How to Implement Data Anonymization in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Anonymization, Privacy

Description: Anonymize MongoDB documents for analytics, testing, and GDPR compliance using aggregation pipelines, pseudonymization, and k-anonymity techniques.

---

## Data Masking vs. Anonymization

Data masking hides real values while preserving format. Anonymization goes further - it permanently transforms data so individuals cannot be identified, even indirectly. True anonymization satisfies GDPR's "no longer personal data" standard. Pseudonymization (replacing identifiers with tokens) reduces risk but is reversible and still counts as personal data.

## Pseudonymization with Consistent Tokens

Replace identifying values with consistent tokens (same input always produces the same output):

```javascript
const crypto = require('crypto')

function pseudonymize(value, secret) {
  return crypto.createHmac('sha256', secret)
    .update(String(value))
    .digest('hex')
    .slice(0, 16)
}

// Pseudonymize a batch of users
const secret = process.env.PSEUDONYM_SECRET  // keep secure, losing it breaks re-linkability

const users = await db.collection('users').find({}).toArray()
const pseudonymized = users.map(u => ({
  ...u,
  userId: pseudonymize(u._id.toString(), secret),
  email: pseudonymize(u.email, secret) + '@anon.example',
  name: 'User_' + pseudonymize(u._id.toString(), secret).slice(0, 6),
  _id: undefined,
}))

await db.collection('users_pseudonymized').insertMany(pseudonymized)
```

## Full Anonymization with Aggregation Pipeline

For irreversible anonymization suitable for analytics exports:

```javascript
db.orders.aggregate([
  // Generalize precise timestamps to month-year only
  {
    $addFields: {
      orderMonth: { $dateToString: { format: "%Y-%m", date: "$createdAt" } },
      // Remove identifying fields
      customerId: "$$REMOVE",
      email: "$$REMOVE",
      ipAddress: "$$REMOVE",
      // Generalize location to region
      region: {
        $switch: {
          branches: [
            { case: { $in: ["$state", ["CA", "OR", "WA"]] }, then: "West" },
            { case: { $in: ["$state", ["NY", "NJ", "CT"]] }, then: "Northeast" },
          ],
          default: "Other",
        },
      },
      // Keep aggregate metrics, remove individual identifiers
      orderValue: {
        $switch: {
          branches: [
            { case: { $lt: ["$total", 50] }, then: "0-49" },
            { case: { $lt: ["$total", 200] }, then: "50-199" },
          ],
          default: "200+",
        },
      },
    },
  },
  { $unset: ["customerId", "email", "ipAddress", "total", "shippingAddress"] },
  { $out: { db: "analytics", coll: "orders_anon" } },
])
```

## K-Anonymity Check

K-anonymity ensures no individual is uniquely identifiable. Each combination of quasi-identifiers (age, ZIP, gender) appears at least k times:

```javascript
async function checkKAnonymity(collection, quasiIdentifiers, k = 5) {
  const groupBy = quasiIdentifiers.reduce((acc, field) => {
    acc[field.replace('.', '_')] = `$${field}`
    return acc
  }, {})

  const results = await db.collection(collection).aggregate([
    { $group: { _id: groupBy, count: { $sum: 1 } } },
    { $match: { count: { $lt: k } } },
    { $sort: { count: 1 } },
    { $limit: 10 },
  ]).toArray()

  if (results.length > 0) {
    console.warn(`K-anonymity violation (k < ${k}):`)
    results.forEach(r => console.warn(r._id, 'count:', r.count))
    return false
  }
  return true
}

await checkKAnonymity('patients_anon', ['ageGroup', 'zipPrefix', 'diagnosis'], 5)
```

## Differential Privacy for Aggregate Queries

Add calibrated noise to aggregate results to prevent re-identification:

```javascript
function addLaplaceNoise(value, sensitivity, epsilon) {
  const scale = sensitivity / epsilon
  const u = Math.random() - 0.5
  return value - scale * Math.sign(u) * Math.log(1 - 2 * Math.abs(u))
}

async function getPrivateCount(filter, sensitivity = 1, epsilon = 0.1) {
  const exact = await db.collection('events').countDocuments(filter)
  return Math.round(addLaplaceNoise(exact, sensitivity, epsilon))
}
```

## Summary

MongoDB data anonymization ranges from reversible pseudonymization (HMAC tokens for re-linkable datasets) to full anonymization via aggregation pipelines that generalize, suppress, and aggregate identifying fields. Verify anonymization quality using k-anonymity checks that ensure no individual is uniquely identifiable by quasi-identifier combinations. For aggregate analytics outputs, add differential privacy noise to prevent inference attacks on small groups.
