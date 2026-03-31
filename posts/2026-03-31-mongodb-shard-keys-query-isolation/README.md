# How to Choose Shard Keys for Query Isolation in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Shard Key, Query, Performance

Description: Learn how to select MongoDB shard keys that route queries to a single shard, avoiding expensive scatter-gather operations across your cluster.

---

## Overview

A well-designed shard key enables targeted queries - queries that MongoDB can route to a single shard because the shard key value (or range) is included in the query filter. When a query does not include the shard key, MongoDB must broadcast the query to all shards (scatter-gather), which is far more expensive and does not scale with cluster size.

## Targeted vs Scatter-Gather Queries

```javascript
// TARGETED: shard key (tenantId) is in the filter - goes to one shard
db.orders.find({ tenantId: 'acme', status: 'pending' });

// SCATTER-GATHER: no shard key - broadcast to ALL shards
db.orders.find({ status: 'pending' });
```

Targeted queries scale linearly with cluster size. Scatter-gather queries get slower as you add shards because there are more shards to query.

## Identifying Your Query Patterns First

Before choosing a shard key, analyze your application's most frequent queries:

```javascript
// Enable profiler to capture queries
db.setProfilingLevel(1, { slowms: 50 });

// Later, review slow queries
db.system.profile.find({ ns: 'shop.orders' }).sort({ ts: -1 }).limit(20)
```

The shard key should appear in the majority of your high-frequency query filters.

## Choosing a Shard Key for Isolation

### Tenant ID (Multi-Tenant Applications)

For multi-tenant SaaS applications, `tenantId` as the shard key ensures all data for a tenant is on one shard and all tenant-scoped queries are targeted:

```javascript
sh.shardCollection('shop.orders', { tenantId: 'hashed' });

// All these queries hit exactly one shard
db.orders.find({ tenantId: 'acme' });
db.orders.find({ tenantId: 'acme', status: 'open' });
db.orders.countDocuments({ tenantId: 'acme' });
```

### User ID (User-Centric Applications)

For social apps or user-data platforms, `userId` ensures all user data lands on the same shard:

```javascript
sh.shardCollection('app.events', { userId: 'hashed' });

// Targeted - all events for a user are on one shard
db.events.find({ userId: '507f1f77bcf86cd799439011' }).sort({ ts: -1 });
```

## Compound Shard Keys for Range + Isolation

If you query by both a category and a date range, a compound shard key can give you both isolation and range efficiency:

```javascript
sh.shardCollection('analytics.events', { region: 1, ts: 1 });

// Targeted to a specific region's shard, with range scan on ts
db.analytics.events.find({
  region: 'us-east',
  ts: { $gte: ISODate('2026-03-01'), $lt: ISODate('2026-04-01') }
});
```

## Using explain() to Verify Targeting

After sharding, verify your queries are targeted:

```javascript
db.orders.find({ tenantId: 'acme', status: 'open' }).explain('executionStats')
```

Look for `"winningPlan"` - if it shows `SINGLE_SHARD` the query is targeted. If it shows `SHARD_MERGE`, it was a scatter-gather.

```javascript
// Quick check via mongos
db.orders.find({ tenantId: 'acme' }).explain().queryPlanner.winningPlan.stage
// "SINGLE_SHARD" = targeted
// "SHARD_MERGE" = scatter-gather
```

## Avoiding Common Mistakes

- **Low cardinality keys** - A boolean or enum with few values results in hotspots on specific shards.
- **Monotonically increasing keys** - Using `createdAt` or `_id` alone sends all new writes to the last chunk (the "right-end" hotspot).
- **Keys that are never in queries** - Any key not in your query predicates forces scatter-gather for every read.

## Summary

The best shard key for query isolation is one that appears in every high-frequency query filter and has high enough cardinality to distribute data evenly. Tenant ID, user ID, or a similar entity identifier are ideal candidates for multi-tenant and user-centric applications. Always verify targeting using `explain()` after sharding, and refine your queries to always include the shard key in the filter.
