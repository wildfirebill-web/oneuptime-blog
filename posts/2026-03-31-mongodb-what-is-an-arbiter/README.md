# What Is an Arbiter in a MongoDB Replica Set

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Arbiter, Replica Set, High Availability, Election

Description: An arbiter is a lightweight MongoDB replica set member that participates in elections but holds no data, used to maintain an odd number of voting members.

---

## Overview

A MongoDB replica set requires a majority of voting members to elect a new primary. If you have two data-bearing members (one primary and one secondary), you have an even number of votes. A network partition or primary failure would leave the remaining member unable to reach a majority and elect itself as primary. An arbiter solves this by adding a third voting member that holds no data and therefore requires minimal resources.

## What an Arbiter Does

- Votes in primary elections to help break ties
- Does not store any data
- Does not serve reads
- Does not become primary
- Has minimal hardware requirements (CPU, memory, disk)

## When to Use an Arbiter

Use an arbiter when:
- You want a three-member replica set for high availability but only have resources for two data nodes
- You need an odd number of votes to prevent split-brain elections
- Adding a full secondary would be cost-prohibitive

**Important note**: MongoDB documentation recommends against using arbiters in production replica sets with three or more data-bearing members. The preferred configuration is three data-bearing members (PSS), not two data nodes plus an arbiter (PSA). The PSA architecture has known drawbacks related to write concern majority and cache pressure.

## Adding an Arbiter

```javascript
// Start a mongod process with --replSet and no data directory requirement
// (or a small one)
// mongod --replSet myReplicaSet --port 27019 --dbpath /data/arb

// Connect to the primary and add the arbiter
rs.addArb("arbiterHost:27019")

// Or with the full config object
rs.add({
  host: "arbiterHost:27019",
  arbiterOnly: true
})
```

## Verifying Arbiter Status

```javascript
// Check replica set configuration
rs.conf()
// Look for: "arbiterOnly": true

// Check replica set status
rs.status()
// The arbiter will show stateStr: "ARBITER"
```

## Arbiter Limitations

Arbiters do not replicate data, so they cannot be promoted to primary. They also cannot be used to satisfy a `w: "majority"` write concern in some configurations. In a PSA replica set with `w: "majority"`, if the secondary goes down, writes will block because only one data node remains and it cannot satisfy majority on its own.

To mitigate this, you can set the secondary's votes to 0 or use a three-node PSS configuration instead.

## Removing an Arbiter

```javascript
// Connect to the primary
rs.remove("arbiterHost:27019")
```

After removal, verify with `rs.status()` that the arbiter no longer appears in the members list.

## Arbiter vs. Non-Voting Secondary

An alternative to an arbiter is a non-voting secondary with `votes: 0`. This member replicates data but does not participate in elections. Unlike an arbiter, it can serve as a warm standby and can even hold data for backups or analytics without affecting the vote count.

## Summary

An arbiter is a lightweight, data-free replica set member used to provide an additional vote in primary elections without the overhead of a full data node. While useful for cost-constrained deployments, the preferred production topology is three data-bearing members. Use arbiters carefully and understand the write concern implications in PSA setups.
