# How to Deploy Apache Cassandra via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Cassandra, NoSQL, Distributed Database, Docker, Wide Column

Description: Learn how to deploy a multi-node Apache Cassandra cluster via Portainer for scalable, distributed wide-column storage.

---

Apache Cassandra is a distributed NoSQL database designed for high write throughput and horizontal scalability. This guide deploys a three-node Cassandra cluster as a Portainer stack.

## Stack Definition

Deploy Cassandra with a seed node and two additional nodes:

```yaml
version: "3.8"

services:
  cassandra1:
    image: cassandra:4.1
    hostname: cassandra1
    environment:
      CASSANDRA_CLUSTER_NAME: "MyCluster"
      CASSANDRA_DC: datacenter1
      CASSANDRA_RACK: rack1
      CASSANDRA_SEEDS: cassandra1
      CASSANDRA_ENDPOINT_SNITCH: GossipingPropertyFileSnitch
      MAX_HEAP_SIZE: "512M"
      HEAP_NEWSIZE: "100M"
    volumes:
      - cassandra1_data:/var/lib/cassandra
    networks:
      - cassandra_net
    healthcheck:
      test: ["CMD-SHELL", "nodetool status | grep -E '^UN'"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 60s

  cassandra2:
    image: cassandra:4.1
    hostname: cassandra2
    environment:
      CASSANDRA_CLUSTER_NAME: "MyCluster"
      CASSANDRA_DC: datacenter1
      CASSANDRA_RACK: rack1
      CASSANDRA_SEEDS: cassandra1
      CASSANDRA_ENDPOINT_SNITCH: GossipingPropertyFileSnitch
      MAX_HEAP_SIZE: "512M"
      HEAP_NEWSIZE: "100M"
    volumes:
      - cassandra2_data:/var/lib/cassandra
    networks:
      - cassandra_net
    depends_on:
      cassandra1:
        condition: service_healthy

  cassandra3:
    image: cassandra:4.1
    hostname: cassandra3
    environment:
      CASSANDRA_CLUSTER_NAME: "MyCluster"
      CASSANDRA_DC: datacenter1
      CASSANDRA_RACK: rack1
      CASSANDRA_SEEDS: cassandra1
      CASSANDRA_ENDPOINT_SNITCH: GossipingPropertyFileSnitch
      MAX_HEAP_SIZE: "512M"
      HEAP_NEWSIZE: "100M"
    volumes:
      - cassandra3_data:/var/lib/cassandra
    networks:
      - cassandra_net
    depends_on:
      cassandra2:
        condition: service_healthy

volumes:
  cassandra1_data:
  cassandra2_data:
  cassandra3_data:

networks:
  cassandra_net:
    driver: bridge
```

Nodes join sequentially to avoid bootstrap conflicts - `cassandra3` waits for `cassandra2`, which waits for `cassandra1`.

## Verifying Cluster Status

Check ring status after all nodes are running:

```bash
docker exec -it $(docker ps -qf name=cassandra1) nodetool status

# Expected output (all nodes show UN = Up/Normal):

# Datacenter: datacenter1
# =======================
# Status=Up/Down
# |/ State=Normal/Leaving/Joining/Moving
# --  Address     Load       Tokens  Owns    Host ID   Rack
# UN  172.x.x.1   75 KiB     16      ?       ...       rack1
# UN  172.x.x.2   51 KiB     16      ?       ...       rack1
# UN  172.x.x.3   51 KiB     16      ?       ...       rack1
```

## Creating a Keyspace and Table

Connect with `cqlsh` and create a keyspace with replication:

```bash
docker exec -it $(docker ps -qf name=cassandra1) cqlsh

-- Create a keyspace with a replication factor of 3
CREATE KEYSPACE IF NOT EXISTS appks
WITH replication = {'class': 'NetworkTopologyStrategy', 'datacenter1': 3};

USE appks;

CREATE TABLE IF NOT EXISTS events (
    event_id   UUID PRIMARY KEY,
    user_id    TEXT,
    action     TEXT,
    created_at TIMESTAMP
);

-- Insert test data
INSERT INTO events (event_id, user_id, action, created_at)
VALUES (uuid(), 'user-123', 'login', toTimestamp(now()));

-- Query
SELECT * FROM events WHERE event_id = <uuid>;
```

## Connecting a Python Application

Use the `cassandra-driver` package:

```python
from cassandra.cluster import Cluster
from cassandra.auth import PlainTextAuthProvider

cluster = Cluster(
    contact_points=['cassandra1', 'cassandra2', 'cassandra3'],
    port=9042
)
session = cluster.connect('appks')

rows = session.execute("SELECT * FROM events LIMIT 10")
for row in rows:
    print(row.event_id, row.user_id, row.action)
```

## Compaction and Repair

Run periodic maintenance to keep the cluster healthy:

```bash
# Run compaction on a table
docker exec -it $(docker ps -qf name=cassandra1) \
  nodetool compact appks events

# Run repair to ensure consistency across nodes
docker exec -it $(docker ps -qf name=cassandra1) \
  nodetool repair appks
```
