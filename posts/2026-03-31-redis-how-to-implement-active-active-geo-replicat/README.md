# How to Implement Active-Active Geo-Replication with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Geo-Replication, Active-Active, CRDT, Multi-Region, High Availability

Description: Learn how to implement active-active geo-replication with Redis Enterprise using CRDTs for conflict-free writes across multiple geographic regions.

---

## What Is Active-Active Geo-Replication?

Active-active geo-replication allows multiple Redis instances in different geographic regions to accept both reads and writes simultaneously. Changes replicate bidirectionally between regions. This differs from standard active-passive replication, where only the primary node accepts writes.

```text
Active-Passive (standard):
  US-East (Primary - reads + writes)
    |
    --> EU-West (Replica - reads only)

Active-Active (CRDT-based):
  US-East (reads + writes)
    <----->
  EU-West (reads + writes)
    <----->
  AP-South (reads + writes)
```

## When to Use Active-Active

- Your users are geographically distributed and need low-latency writes
- You require regional failover without traffic cutover delays
- Data can tolerate eventual consistency with CRDT conflict resolution

## Active-Active in Redis Enterprise

Redis Community Edition does not support active-active geo-replication. This feature is exclusive to Redis Enterprise and Redis Cloud.

Redis Enterprise implements active-active using CRDTs (Conflict-free Replicated Data Types). CRDT rules ensure that concurrent writes to the same key from different regions converge to a deterministic value without requiring coordination.

## Setting Up Active-Active with Redis Enterprise REST API

```bash
# Create an active-active database across two clusters
# This uses the Redis Enterprise REST API

curl -k -u admin@myorg.com:password \
  -X POST https://cluster1.myorg.com:9443/v1/bdbs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-active-active-db",
    "type": "redis",
    "replication": true,
    "sync": "distributed",
    "shards_count": 3,
    "memory_size": 4294967296,
    "crdt": true,
    "crdt_sync_nets": [
      {
        "uid": 1,
        "url": "redis://cluster2.myorg.com:5000"
      }
    ]
  }'
```

## CRDT Data Types and Conflict Resolution

Redis Enterprise uses CRDT variants of standard Redis data types:

```text
Data Type       CRDT Behavior
---------       -------------
String          LWW (Last Write Wins) by timestamp
Counter (INCRBY) Increment merges: all increments from all regions are summed
Set             Union: members added in any region appear in all regions
Hash field      LWW per field by timestamp
Sorted Set      Score is the max across regions for the same member
```

### Counter Example (Conflict-Free)

```javascript
const Redis = require('ioredis');

// Region 1 (US-East)
const redisUS = new Redis({ host: 'redis-us-east.myorg.com', port: 6379 });

// Region 2 (EU-West)
const redisEU = new Redis({ host: 'redis-eu-west.myorg.com', port: 6379 });

// Both regions increment the same counter concurrently
// CRDT guarantees: final value = sum of all increments
await redisUS.incrby('page:views:homepage', 5);  // US sees 5 new views
await redisEU.incrby('page:views:homepage', 3);  // EU sees 3 new views

// After replication converges:
// Both regions will show 8 (5 + 3 = 8)
```

### Set Example (Union Semantics)

```javascript
// Region 1: add user to online set
await redisUS.sadd('users:online', 'user-42');

// Region 2: add different user
await redisEU.sadd('users:online', 'user-99');

// After convergence: both regions show {'user-42', 'user-99'}
```

### String Conflict (Last Write Wins)

```javascript
// Concurrent string writes - ONLY ONE WINS
// Determined by timestamp at write time

// US writes at t=1000
await redisUS.set('config:feature-flag', 'enabled');

// EU writes at t=1001 (slightly later)
await redisEU.set('config:feature-flag', 'disabled');

// After convergence: 'disabled' wins (higher timestamp)
// LWW means one region's write is discarded
```

## Simulating Active-Active with Community Edition

Community Edition does not support true active-active. You can simulate limited scenarios using custom application logic:

```python
import redis
import time

# Two Redis instances (acting as "regions")
r_us = redis.Redis(host='redis-us', port=6379, decode_responses=True)
r_eu = redis.Redis(host='redis-eu', port=6379, decode_responses=True)

def crdt_counter_increment(key: str, amount: int):
    """Increment in both regions - application-level CRDT."""
    timestamp = int(time.time() * 1000)

    r_us.incrby(key, amount)
    r_eu.incrby(key, amount)

def crdt_set_add(key: str, member: str):
    """Add to set in both regions - union semantics."""
    r_us.sadd(key, member)
    r_eu.sadd(key, member)

def sync_regions_for_key(key: str):
    """Manual sync: merge both regions (for set union)."""
    us_members = r_us.smembers(key)
    eu_members = r_eu.smembers(key)

    all_members = us_members | eu_members
    if all_members:
        r_us.sadd(key, *all_members)
        r_eu.sadd(key, *all_members)
```

## Measuring Replication Lag

```bash
# In Redis Enterprise, check replication status
redis-cli -h redis-us-east.myorg.com INFO replication

# Key fields:
# role: master
# connected_slaves: 1
# slave0: offset=12345,lag=0  <- lag in seconds between regions
```

## Routing Reads and Writes

```javascript
class GeoRedisClient {
  constructor(regions) {
    this.clients = regions.map(r => new Redis({ host: r.host, port: r.port }));
    this.localIndex = 0; // Index of closest region
  }

  // Writes go to local region (replicated to others)
  async write(command, ...args) {
    return this.clients[this.localIndex][command](...args);
  }

  // Reads from local region (low latency)
  async read(command, ...args) {
    return this.clients[this.localIndex][command](...args);
  }
}

const geoClient = new GeoRedisClient([
  { host: 'redis-us-east.myorg.com', port: 6379 },
  { host: 'redis-eu-west.myorg.com', port: 6379 }
]);
```

## Designing for Active-Active Compatibility

```text
Use active-active friendly operations:
  - INCRBY / DECRBY (CRDT counters)
  - SADD / SREM (CRDT sets)
  - Append-only patterns

Avoid or handle carefully:
  - SET on shared keys (LWW, one write discarded)
  - Compare-and-swap patterns (WATCH/MULTI may not work cross-region)
  - Read-modify-write patterns (not atomic cross-region)
```

## Summary

Active-active geo-replication enables low-latency writes in multiple regions simultaneously, powered by CRDT conflict resolution in Redis Enterprise. Counters (INCRBY) and Sets (SADD) are naturally CRDT-compatible. String writes use last-write-wins semantics, so avoid concurrent string updates to the same key from multiple regions. Design your data model around CRDT-friendly operations to get the most benefit from active-active replication.
