# How to Set Up ClickHouse Replication with ZooKeeper

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ZooKeeper, Replication, Database, Distributed System

Description: Learn how to configure ClickHouse replication with ZooKeeper step by step, including cluster config, ZooKeeper setup, and ReplicatedMergeTree table creation.

ClickHouse replication lets you keep multiple copies of your data across different servers so that reads scale out and a single node failure does not cause data loss. ZooKeeper acts as the coordination layer - it stores replication metadata, tracks which replicas have which data parts, and coordinates merges. This guide walks through a complete setup from scratch.

## Architecture Overview

A replicated ClickHouse cluster needs three components:

1. A ZooKeeper ensemble (3 nodes for production, 1 for development)
2. Two or more ClickHouse servers with matching cluster config
3. Tables created with a `ReplicatedMergeTree` engine family

ZooKeeper does not store the actual data. It stores only metadata: the list of parts each replica has, pending mutations, merge tasks, and replica states. The actual data lives on the ClickHouse servers and is transferred directly between replicas.

## Step 1: Install and Configure ZooKeeper

Install ZooKeeper on three dedicated nodes. Using the system package is the quickest path:

```bash
apt-get install -y zookeeper zookeeperd
```

Configure each ZooKeeper node. The `myid` file and `zoo.cfg` must match:

```bash
# On node 1
echo "1" > /var/lib/zookeeper/myid

# On node 2
echo "2" > /var/lib/zookeeper/myid

# On node 3
echo "3" > /var/lib/zookeeper/myid
```

Create `/etc/zookeeper/conf/zoo.cfg` on all three nodes:

```text
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=60
autopurge.snapRetainCount=3
autopurge.purgeInterval=1

server.1=zk1.internal:2888:3888
server.2=zk2.internal:2888:3888
server.3=zk3.internal:2888:3888
```

Start ZooKeeper on all three nodes and verify the ensemble is healthy:

```bash
systemctl start zookeeper
systemctl enable zookeeper

# Check which node is the leader
echo "stat" | nc zk1.internal 2181 | grep Mode
echo "stat" | nc zk2.internal 2181 | grep Mode
echo "stat" | nc zk3.internal 2181 | grep Mode
```

You should see one node reporting `Mode: leader` and the others `Mode: follower`.

## Step 2: Configure ClickHouse to Use ZooKeeper

On every ClickHouse server, create `/etc/clickhouse-server/config.d/zookeeper.xml`:

```xml
<clickhouse>
    <zookeeper>
        <node>
            <host>zk1.internal</host>
            <port>2181</port>
        </node>
        <node>
            <host>zk2.internal</host>
            <port>2181</port>
        </node>
        <node>
            <host>zk3.internal</host>
            <port>2181</port>
        </node>
        <session_timeout_ms>30000</session_timeout_ms>
        <operation_timeout_ms>10000</operation_timeout_ms>
        <root>/clickhouse</root>
        <identity>clickhouse:secret</identity>
    </zookeeper>
</clickhouse>
```

The `<root>` element scopes all ClickHouse ZooKeeper paths under `/clickhouse`. This is important if you share the ZooKeeper ensemble with other services. The `<identity>` element enables ZooKeeper ACL authentication.

## Step 3: Define the Cluster in ClickHouse Config

Create `/etc/clickhouse-server/config.d/cluster.xml` on all ClickHouse nodes:

```xml
<clickhouse>
    <remote_servers>
        <production_cluster>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch1.internal</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>secret</password>
                </replica>
                <replica>
                    <host>ch2.internal</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>secret</password>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch3.internal</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>secret</password>
                </replica>
                <replica>
                    <host>ch4.internal</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>secret</password>
                </replica>
            </shard>
        </production_cluster>
    </remote_servers>
</clickhouse>
```

Set `<internal_replication>true</internal_replication>` so that when you insert into a `Distributed` table, ClickHouse sends the data to one replica per shard and lets the replication mechanism propagate it to the other replicas. Setting it to `false` would cause the `Distributed` table to write to all replicas itself, which is less efficient.

## Step 4: Set the Replica Macros

Each ClickHouse node needs a unique identity for the ZooKeeper paths. Create `/etc/clickhouse-server/config.d/macros.xml` - the values must differ per node:

```xml
<!-- On ch1.internal (shard 1, replica 1) -->
<clickhouse>
    <macros>
        <cluster>production_cluster</cluster>
        <shard>01</shard>
        <replica>ch1.internal</replica>
    </macros>
</clickhouse>
```

```xml
<!-- On ch2.internal (shard 1, replica 2) -->
<clickhouse>
    <macros>
        <cluster>production_cluster</cluster>
        <shard>01</shard>
        <replica>ch2.internal</replica>
    </macros>
</clickhouse>
```

```xml
<!-- On ch3.internal (shard 2, replica 1) -->
<clickhouse>
    <macros>
        <cluster>production_cluster</cluster>
        <shard>02</shard>
        <replica>ch3.internal</replica>
    </macros>
</clickhouse>
```

## Step 5: Create a Replicated Table

Reload ClickHouse config and create a table using the macros:

```bash
systemctl reload clickhouse-server
```

```sql
CREATE TABLE events ON CLUSTER production_cluster
(
    event_date  Date,
    event_time  DateTime,
    user_id     UInt64,
    event_type  LowCardinality(String),
    properties  String
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',
    '{replica}'
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_type, event_date, user_id)
TTL event_date + INTERVAL 1 YEAR;
```

The ZooKeeper path `/clickhouse/tables/{shard}/events` becomes `/clickhouse/tables/01/events` on shard 1 nodes. Both replicas in the same shard share this path, which is how ClickHouse knows they are replicas of each other. The `{replica}` macro becomes the unique replica name within that path.

## Step 6: Verify Replication is Working

Insert data on one replica and confirm it appears on the other:

```sql
-- Insert on ch1.internal
INSERT INTO events VALUES
    (today(), now(), 1001, 'page_view', '{"page": "/home"}'),
    (today(), now(), 1002, 'click',     '{"element": "signup"}');

-- Query on ch2.internal (the other replica)
SELECT count() FROM events;
```

Check replication status in the system tables:

```sql
-- Check replica status on any node
SELECT
    database,
    table,
    is_leader,
    is_readonly,
    absolute_delay,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    last_queue_update,
    zookeeper_path
FROM system.replicas
WHERE table = 'events';
```

`absolute_delay` is the most important field. A value of 0 means the replica is fully caught up. A growing value means replication is falling behind.

## Common Configuration Mistakes

**Wrong ZooKeeper path for different tables**: Every table must have its own unique ZooKeeper path. Using the same path for two different tables corrupts the replication state.

**Identical `{replica}` macros on two nodes in the same shard**: If two replicas have the same replica name in ZooKeeper, they will conflict. Always use the hostname or a unique identifier.

**Forgetting to set `<internal_replication>true`**: Without this, your `Distributed` table will double-write to both replicas, causing duplicates.

**Using ZooKeeper without the `<root>` scoping**: If you share a ZooKeeper ensemble across environments, missing `<root>` causes staging and production to share paths and corrupt each other's state.

Once your replicated tables are running, monitor `system.replicas` and `system.replication_queue` regularly. The replication queue shows pending work and is the first place to look when replication falls behind.
