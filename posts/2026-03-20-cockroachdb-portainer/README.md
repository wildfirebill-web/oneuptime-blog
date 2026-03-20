# How to Deploy CockroachDB via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, CockroachDB, Distributed Database, PostgreSQL, SQL

Description: Deploy a CockroachDB distributed SQL cluster using Portainer for globally distributed, PostgreSQL-compatible database workloads.

## Introduction

CockroachDB is a distributed SQL database compatible with PostgreSQL. It provides horizontal scaling, automatic replication, and survives node failures without downtime. This guide covers deploying a 3-node CockroachDB cluster using Portainer.

## Step 1: Deploy CockroachDB Cluster

```yaml
# docker-compose.yml - CockroachDB 3-node cluster
version: "3.8"

networks:
  cockroach_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.32.0.0/24

volumes:
  crdb1_data:
  crdb2_data:
  crdb3_data:

services:
  # CockroachDB Node 1
  crdb1:
    image: cockroachdb/cockroach:v23.2.0
    container_name: crdb1
    restart: unless-stopped
    hostname: crdb1
    networks:
      cockroach_net:
        ipv4_address: 172.32.0.10
    ports:
      - "26257:26257"  # SQL port
      - "8080:8080"    # Admin UI
    volumes:
      - crdb1_data:/cockroach/cockroach-data
    command: >
      start
      --insecure
      --advertise-addr=crdb1
      --join=crdb1:26257,crdb2:26257,crdb3:26257
      --cache=256MiB
      --max-sql-memory=256MiB
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health?ready=1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # CockroachDB Node 2
  crdb2:
    image: cockroachdb/cockroach:v23.2.0
    container_name: crdb2
    restart: unless-stopped
    hostname: crdb2
    networks:
      cockroach_net:
        ipv4_address: 172.32.0.11
    ports:
      - "26258:26257"
      - "8081:8080"
    volumes:
      - crdb2_data:/cockroach/cockroach-data
    command: >
      start
      --insecure
      --advertise-addr=crdb2
      --join=crdb1:26257,crdb2:26257,crdb3:26257
      --cache=256MiB
      --max-sql-memory=256MiB

  # CockroachDB Node 3
  crdb3:
    image: cockroachdb/cockroach:v23.2.0
    container_name: crdb3
    restart: unless-stopped
    hostname: crdb3
    networks:
      cockroach_net:
        ipv4_address: 172.32.0.12
    ports:
      - "26259:26257"
      - "8082:8080"
    volumes:
      - crdb3_data:/cockroach/cockroach-data
    command: >
      start
      --insecure
      --advertise-addr=crdb3
      --join=crdb1:26257,crdb2:26257,crdb3:26257
      --cache=256MiB
      --max-sql-memory=256MiB

  # CockroachDB cluster initializer
  crdb_init:
    image: cockroachdb/cockroach:v23.2.0
    container_name: crdb_init
    restart: "no"
    networks:
      - cockroach_net
    command: >
      init
      --insecure
      --host=crdb1:26257
    depends_on:
      crdb1:
        condition: service_healthy
```

## Step 2: Initialize and Configure the Cluster

```bash
# After stack deployment, check cluster status
docker exec crdb1 ./cockroach node status --insecure

# Create application database and user
docker exec crdb1 ./cockroach sql --insecure --execute="
CREATE DATABASE myapp;
CREATE USER appuser WITH PASSWORD 'app_secure_password';
GRANT ALL ON DATABASE myapp TO appuser;
"

# Set cluster-wide settings
docker exec crdb1 ./cockroach sql --insecure --execute="
SET CLUSTER SETTING sql.telemetry.query_sampling.enabled = false;
SET CLUSTER SETTING cluster.organization = 'My Company';
SET CLUSTER SETTING enterprise.license = '';
"
```

## Step 3: Connect Applications (PostgreSQL-Compatible)

```python
# Python - CockroachDB is PostgreSQL-compatible
import psycopg2
from urllib.parse import urlparse

# Connect to any node (all are equal in CockroachDB)
conn = psycopg2.connect(
    host="crdb1",
    port=26257,
    database="myapp",
    user="appuser",
    password="app_secure_password",
    # Enable SSL for production
    sslmode="disable"  # Use "require" in production with certs
)

cursor = conn.cursor()

# Create a table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name STRING NOT NULL,
        email STRING UNIQUE NOT NULL,
        created_at TIMESTAMPTZ DEFAULT now()
    )
""")

# Insert with CockroachDB-specific types
cursor.execute(
    "INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id",
    ("Alice", "alice@example.com")
)
user_id = cursor.fetchone()[0]
conn.commit()

print(f"Created user with ID: {user_id}")
```

## Step 4: CockroachDB-Specific Features

```sql
-- CockroachDB geospatial data
CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING,
    coords GEOGRAPHY(POINT, 4326)
);

INSERT INTO locations (name, coords) VALUES (
    'New York',
    ST_MakePoint(-74.0060, 40.7128)
);

-- Find locations within 100km of a point
SELECT name FROM locations
WHERE ST_DWithin(coords, ST_MakePoint(-74.0060, 40.7128)::geography, 100000);

-- Multi-region table configuration
ALTER TABLE users SET LOCALITY GLOBAL;

-- Table partitioned across regions
ALTER TABLE orders SET LOCALITY REGIONAL BY ROW;
```

## Step 5: Monitor CockroachDB

```bash
# Check cluster health
docker exec crdb1 ./cockroach node status --insecure

# View database metrics
curl http://localhost:8080/_status/nodes | jq '.nodes[].desc.nodeId'

# Check for range issues
docker exec crdb1 ./cockroach debug zip /tmp/debug.zip --insecure

# View SQL statistics
docker exec crdb1 ./cockroach sql --insecure --execute="
SELECT * FROM crdb_internal.cluster_queries
ORDER BY start DESC LIMIT 10;
"
```

## Step 6: Production Considerations

```yaml
# Production configuration with TLS
services:
  crdb1:
    command: >
      start
      --certs-dir=/cockroach/certs
      --advertise-addr=crdb1
      --join=crdb1:26257,crdb2:26257,crdb3:26257
      --listen-addr=0.0.0.0
      --http-addr=0.0.0.0:8080
    volumes:
      - crdb1_data:/cockroach/cockroach-data
      - /opt/certs:/cockroach/certs:ro
```

```bash
# Generate TLS certificates
docker run --rm -v /opt/certs:/certs cockroachdb/cockroach cert create-ca \
  --certs-dir=/certs --ca-key=/certs/ca.key

docker run --rm -v /opt/certs:/certs cockroachdb/cockroach cert create-node \
  crdb1 crdb2 crdb3 localhost 127.0.0.1 \
  --certs-dir=/certs --ca-key=/certs/ca.key
```

## Conclusion

CockroachDB gives you a horizontally scalable, PostgreSQL-compatible database that can survive node failures without data loss. The distributed architecture means writes can go to any node and are automatically replicated. Portainer makes managing the multi-container cluster straightforward, with visibility into each node's health and logs. The Admin UI at port 8080 provides detailed metrics on query performance, replication, and cluster health.
