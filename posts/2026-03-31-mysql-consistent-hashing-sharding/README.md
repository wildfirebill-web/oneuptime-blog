# How to Implement Consistent Hashing for MySQL Sharding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Sharding, Consistent Hashing, Scaling, Architecture

Description: Learn how to use consistent hashing for MySQL sharding to minimize data migration when adding or removing shards as your database scales.

---

## Why Consistent Hashing?

Simple modulo-based sharding (e.g., `user_id % 4`) works well until you need to add a new shard. When the shard count changes from 4 to 5, `user_id % 5` routes nearly all rows to different shards, requiring a full data migration.

Consistent hashing solves this by distributing shard keys on a virtual ring. When a shard is added or removed, only the fraction of keys near that shard's ring position needs to move - typically `1/N` of total data, where N is the number of shards.

## How the Hash Ring Works

The consistent hash ring is a circular space of values (e.g., 0 to 2^32-1). Each shard occupies one or more positions on the ring (virtual nodes). To find a key's shard, hash the key and walk the ring clockwise to the next shard position:

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, shards: list, virtual_nodes: int = 150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        for shard in shards:
            self.add_shard(shard)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_shard(self, shard: str):
        for i in range(self.virtual_nodes):
            vnode_key = self._hash(f"{shard}:{i}")
            self.ring[vnode_key] = shard
            bisect.insort(self.sorted_keys, vnode_key)

    def remove_shard(self, shard: str):
        for i in range(self.virtual_nodes):
            vnode_key = self._hash(f"{shard}:{i}")
            del self.ring[vnode_key]
            self.sorted_keys.remove(vnode_key)

    def get_shard(self, key: str) -> str:
        if not self.ring:
            raise ValueError("No shards in ring")
        hash_val = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, hash_val) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]
```

## Setting Up Shards

Initialize the ring with your MySQL shard hosts:

```python
import mysql.connector

shards = ["shard0.db", "shard1.db", "shard2.db", "shard3.db"]
ring = ConsistentHashRing(shards)

connections = {
    host: mysql.connector.connect(
        host=host, user="app", password="secret", database="mydb"
    )
    for host in shards
}

def get_conn(user_id: int):
    shard_host = ring.get_shard(str(user_id))
    return connections[shard_host]
```

## Routing Queries

```python
def get_user(user_id: int):
    conn = get_conn(user_id)
    cursor = conn.cursor(dictionary=True)
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    return cursor.fetchone()
```

## Adding a New Shard

When capacity grows, add a new shard to the ring. Only the keys between the new shard's position and its predecessor need to be migrated:

```python
# Step 1: Add new shard to ring (in-memory only)
new_shard = "shard4.db"
connections[new_shard] = mysql.connector.connect(
    host=new_shard, user="app", password="secret", database="mydb"
)

# Step 2: Identify which existing keys now route to the new shard
ring.add_shard(new_shard)

# Step 3: Migrate affected rows (simplified example)
for user_id in range(1, 10_000_000):
    shard = ring.get_shard(str(user_id))
    if shard == new_shard:
        # Copy row from old shard to new shard, then delete
        pass
```

In production, use Vitess or a dedicated resharding tool for zero-downtime migrations.

## Storing Shard Assignment for Immutable Keys

For immutable shard assignments (keys that should never move), store the shard assignment in a lookup table:

```sql
CREATE TABLE shard_assignments (
  entity_id BIGINT UNSIGNED NOT NULL,
  shard_host VARCHAR(255) NOT NULL,
  assigned_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (entity_id)
) ENGINE=InnoDB;
```

Look up this table before routing to avoid re-hashing:

```python
def get_shard_for_entity(entity_id: int) -> str:
    cursor = meta_conn.cursor(dictionary=True)
    cursor.execute(
        "SELECT shard_host FROM shard_assignments WHERE entity_id = %s",
        (entity_id,)
    )
    row = cursor.fetchone()
    if row:
        return row["shard_host"]
    # New entity - assign via ring and persist
    shard = ring.get_shard(str(entity_id))
    cursor.execute(
        "INSERT INTO shard_assignments (entity_id, shard_host) VALUES (%s, %s)",
        (entity_id, shard)
    )
    meta_conn.commit()
    return shard
```

## Summary

Consistent hashing minimizes data migration when rescaling MySQL shards by distributing keys on a virtual ring. Implement a hash ring with virtual nodes for even distribution, route queries by hashing the shard key, and add new shards by migrating only the affected key range. For production, persist shard assignments in a lookup table to avoid re-routing issues and pair with tools like Vitess for automated resharding.
