# How to Use Redis Cloud Fixed Plans vs Flexible Plans

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis Cloud, Pricing, Capacity Planning, Cloud

Description: Understand the difference between Redis Cloud Fixed and Flexible plans to pick the right tier for your workload, team size, and budget.

---

Redis Cloud offers two main subscription types: **Fixed** and **Flexible** (previously called "Annual" or "Pay-as-you-go"). Choosing between them affects cost, resource limits, and operational flexibility. This post explains how each works and when to use one over the other.

## Fixed Plans

Fixed plans provision a dedicated, pre-sized database. You pay a flat monthly rate regardless of actual usage.

Key characteristics:
- **Predictable cost** - flat rate per month.
- **Dedicated resources** - no noisy neighbors.
- **Limited scalability** - you must upgrade the plan to get more RAM.
- **Available sizes**: from 250 MB (free tier) up to 12 GB.

Use Fixed plans when:
- You have a stable, well-understood workload.
- You want a predictable bill with no surprises.
- Your team is small or the database is non-critical.

```text
Free tier:  30 MB  | $0/month
250 MB:      250 MB | ~$7/month
1 GB:        1 GB   | ~$20/month
2.5 GB:      2.5 GB | ~$47/month
```

## Flexible Plans

Flexible plans let you configure exact memory, throughput, replication, and module requirements. You are billed based on what you provision.

Key characteristics:
- **Granular control** - set exact memory, throughput (ops/sec), and replication factor.
- **Redis Stack modules** - enable RediSearch, RedisJSON, RedisGraph, RedisTimeSeries.
- **Active-Active geo-distribution** available.
- **Scales up or down** without plan migration.

Use Flexible plans when:
- You need Redis Stack modules (RediSearch, RedisJSON, etc.).
- Your workload is large (more than 12 GB) or highly variable.
- You need multi-region or Active-Active replication.
- You want fine-grained throughput control (committed ops/sec).

## Comparing the Two

| Feature | Fixed | Flexible |
|---------|-------|----------|
| Max memory | 12 GB | Hundreds of GB |
| Redis Stack modules | No | Yes |
| Active-Active | No | Yes |
| Pricing model | Flat monthly | Provisioned usage |
| Scaling | Plan upgrade required | Adjust in-place |
| Best for | Small/stable workloads | Production at scale |

## Migrating from Fixed to Flexible

When you outgrow a Fixed plan:

1. In the Redis Cloud console, create a new Flexible subscription.
2. Export your data using `redis-cli --rdb dump.rdb`.
3. Import into the Flexible database using `redis-cli --pipe`.
4. Update your application's connection string.
5. Delete the Fixed subscription.

Alternatively, use `REPLICAOF` to live-migrate:

```bash
# On the new Flexible database
redis-cli -h <flexible-host> -p <port> -a <password> \
  REPLICAOF <fixed-host> <fixed-port>

# After sync completes, promote and point app to new host
redis-cli -h <flexible-host> -p <port> -a <password> \
  REPLICAOF NO ONE
```

## Cost Estimation for Flexible Plans

For a Flexible database with:
- 1 GB memory
- 1,000 ops/sec throughput
- Replication enabled (1 replica)

The monthly cost is approximately $60-80 depending on region. Use the Redis Cloud pricing calculator at [redis.io/pricing](https://redis.io/pricing) for exact figures.

## Summary

Fixed plans are the right choice for small, predictable workloads where cost simplicity matters most. Flexible plans unlock Redis Stack modules, Active-Active geo-distribution, and in-place scaling for production applications. Start with Fixed for development and migrate to Flexible when you need modules or your memory requirements exceed 12 GB.
