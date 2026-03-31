# How to Store Subscription and Billing Data in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Subscription, Billing, Schema Design, SaaS

Description: Design MongoDB schemas for subscription and billing data in SaaS applications, covering plan definitions, subscriber records, billing cycles, and invoice storage.

---

## Schema Design Principles for Billing

Billing data requires careful schema design because:
- Plans change over time but existing subscriptions must preserve their original terms
- Invoices are immutable financial records once issued
- Billing cycle calculations must be deterministic and auditable
- Upgrades, downgrades, and proration need clear data lineage

## Plan Definition Schema

Store plan definitions separately from subscriptions. Subscriptions reference a plan by ID but embed key terms at subscription time:

```javascript
// plans collection - source of truth for pricing
{
  _id: "plan_pro_monthly",
  name: "Pro Monthly",
  interval: "month",
  intervalCount: 1,
  amount: 2900,        // $29.00 in cents
  currency: "USD",
  features: ["unlimited_users", "priority_support", "api_access"],
  createdAt: ISODate("2025-01-01T00:00:00Z"),
  active: true,
  metadata: { internalCode: "PRO_M" }
}
```

## Subscription Document Schema

```javascript
// subscriptions collection
{
  _id: "sub_01HW5X2N3M",
  customerId: "cust_abc123",
  planId: "plan_pro_monthly",

  // Snapshot plan terms at subscription creation time
  billingInterval: "month",
  billingIntervalCount: 1,
  unitAmount: 2900,
  currency: "USD",

  status: "active",    // trialing | active | past_due | cancelled | paused

  currentPeriodStart: ISODate("2026-03-01T00:00:00Z"),
  currentPeriodEnd: ISODate("2026-04-01T00:00:00Z"),
  trialEnd: null,

  cancelAt: null,          // Scheduled cancellation date
  cancelledAt: null,       // Actual cancellation timestamp
  cancelReason: null,

  paymentMethodId: "pm_card_xyz",
  collectionMethod: "charge_automatically",

  // Proration and credits
  balance: 0,              // Customer credit balance in cents

  createdAt: ISODate("2026-01-01T00:00:00Z"),
  metadata: {}
}
```

## Invoice Schema

```javascript
// invoices collection - immutable once finalized
{
  _id: "inv_01HW5X2N3M",
  subscriptionId: "sub_01HW5X2N3M",
  customerId: "cust_abc123",
  number: "INV-2026-0142",

  status: "paid",      // draft | open | paid | void | uncollectible

  // Billing period this invoice covers
  periodStart: ISODate("2026-03-01T00:00:00Z"),
  periodEnd: ISODate("2026-04-01T00:00:00Z"),

  lines: [
    {
      description: "Pro Monthly x 1",
      quantity: 1,
      unitAmount: 2900,
      amount: 2900,
      type: "subscription"
    }
  ],

  subtotal: 2900,
  tax: 261,            // 9% tax
  total: 3161,

  dueDate: ISODate("2026-03-15T00:00:00Z"),
  paidAt: ISODate("2026-03-01T00:00:01Z"),
  transactionId: "txn_xyz",

  createdAt: ISODate("2026-03-01T00:00:00Z"),
  finalizedAt: ISODate("2026-03-01T00:00:00Z")
}
```

## Key Indexes

```javascript
// Customer billing dashboard
db.subscriptions.createIndex({ customerId: 1, status: 1 })

// Renewal processing - find subscriptions due for billing
db.subscriptions.createIndex({ currentPeriodEnd: 1, status: 1 })

// Invoice lookup
db.invoices.createIndex({ subscriptionId: 1, createdAt: -1 })
db.invoices.createIndex({ customerId: 1, status: 1, createdAt: -1 })
db.invoices.createIndex({ number: 1 }, { unique: true })
```

## Finding Subscriptions Due for Renewal

```javascript
async function getSubscriptionsDueForRenewal() {
  const now = new Date();
  const in24h = new Date(Date.now() + 24 * 3600 * 1000);

  return db.collection("subscriptions").find({
    status: "active",
    currentPeriodEnd: { $gte: now, $lte: in24h }
  }).toArray();
}
```

## Summary

Subscription and billing data in MongoDB requires three core collections: plans (pricing definitions), subscriptions (customer plan enrollments with embedded billing terms), and invoices (immutable financial records per billing cycle). Embed key plan terms in the subscription at creation time so plan price changes don't affect existing subscribers. Index subscriptions on `currentPeriodEnd + status` to efficiently find renewals due for processing. Treat invoices as append-only records and never modify `total` or `lines` after finalization.
