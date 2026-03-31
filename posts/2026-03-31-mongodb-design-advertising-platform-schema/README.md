# How to Design an Advertising Platform Schema in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema, Design, Advertisement

Description: Learn how to design a scalable MongoDB schema for an advertising platform covering campaigns, ad creatives, targeting rules, and impression tracking.

---

## Core Entities

An advertising platform involves advertisers, campaigns, ad creatives, targeting rules, and impression/click events. MongoDB's flexible document model suits this domain well because targeting rules vary by campaign and creative metadata varies by ad format.

## Advertiser Collection

```json
{
  "_id": "adv-001",
  "name": "Acme Corp",
  "contactEmail": "ads@acme.com",
  "billingInfo": {
    "currency": "USD",
    "paymentMethod": "credit_card",
    "creditLimitCents": 500000
  },
  "status": "active",
  "createdAt": "2026-01-01T00:00:00Z"
}
```

## Campaign Collection

```json
{
  "_id": "camp-001",
  "advertiserId": "adv-001",
  "name": "Spring Sale 2026",
  "objective": "conversions",
  "status": "active",
  "budget": {
    "totalCents": 1000000,
    "dailyCents": 50000,
    "spentCents": 120000
  },
  "schedule": {
    "startDate": "2026-03-01T00:00:00Z",
    "endDate":   "2026-03-31T23:59:59Z"
  },
  "targeting": {
    "geoCountries": ["US", "CA"],
    "geoRegions": ["CA", "NY"],
    "ageRange": { "min": 25, "max": 45 },
    "interests": ["sports", "fitness"],
    "deviceTypes": ["mobile", "desktop"]
  },
  "createdAt": "2026-02-15T00:00:00Z"
}
```

Embedding targeting rules in the campaign document avoids a join on every ad serving request.

## Ad Creative Collection

```json
{
  "_id": "creative-001",
  "campaignId": "camp-001",
  "advertiserId": "adv-001",
  "format": "banner_300x250",
  "headline": "Spring Sale - 30% Off!",
  "imageUrl": "https://cdn.acme.com/ads/spring2026.jpg",
  "destinationUrl": "https://acme.com/sale?utm_campaign=spring2026",
  "status": "approved",
  "qualityScore": 8.5,
  "createdAt": "2026-02-20T00:00:00Z"
}
```

## Impression Events Collection (Bucket Pattern)

Ad platforms generate billions of impressions. Use the Bucket Pattern to group events by creative and hour:

```javascript
{
  _id: ObjectId(),
  creativeId:  "creative-001",
  campaignId:  "camp-001",
  advertiserId: "adv-001",
  hour: ISODate("2026-03-15T14:00:00Z"),
  impressions: 3500,
  clicks: 42,
  conversions: 3,
  spendCents: 8400,
  events: [
    {
      ts: ISODate("2026-03-15T14:01:22Z"),
      userId: "u-xyz",
      action: "click",
      device: "mobile"
    }
    // ... raw events sampled
  ]
}
```

Aggregated counts (`impressions`, `clicks`, `spendCents`) are updated atomically with `$inc`. Raw events are sampled to a bounded array.

## Key Indexes

```javascript
// Campaign lookup by advertiser and status
db.campaigns.createIndex({ advertiserId: 1, status: 1 })

// Ad serving: find active campaigns matching a date range
db.campaigns.createIndex({ status: 1, "schedule.startDate": 1, "schedule.endDate": 1 })

// Creative lookup by campaign
db.creatives.createIndex({ campaignId: 1, status: 1 })

// Reporting: aggregate impressions by campaign and hour
db.impressionBuckets.createIndex({ campaignId: 1, hour: -1 })
```

## Reporting Aggregation

Sum spend and impressions per campaign for the last 7 days:

```javascript
db.impressionBuckets.aggregate([
  {
    $match: {
      hour: { $gte: new Date(Date.now() - 7 * 24 * 3600 * 1000) }
    }
  },
  {
    $group: {
      _id: "$campaignId",
      totalImpressions: { $sum: "$impressions" },
      totalClicks:      { $sum: "$clicks" },
      totalSpendCents:  { $sum: "$spendCents" }
    }
  }
])
```

## Summary

Embed targeting rules within campaign documents to eliminate per-request joins on the ad serving hot path. Store ad creative metadata in a separate collection for independent lifecycle management. Use the Bucket Pattern with pre-aggregated counters for impression and click data to handle high write throughput. Index campaigns on `status` and schedule dates for efficient ad selection queries.
