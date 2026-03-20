# How to Deploy CockroachDB via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, CockroachDB, Distributed SQL, Database, High Availability, Docker

Description: Learn how to deploy a multi-node CockroachDB cluster via Portainer for distributed SQL with automatic replication and failover.

---

CockroachDB is a distributed SQL database that replicates data automatically across nodes. This guide deploys a three-node CockroachDB cluster as a Portainer stack.

## Stack Definition

```yaml
version: "3.8"

services:
  roach1:
    image: cockroachdb/cockroach:v23.2.0
    hostname: roach1
    command: start --insecure --join=roach1,roach2,roach3 --listen-addr=roach1:26257 --advertise-addr=roach1:26257 --http-addr=roach1:8080
    volumes:
      - roach1_data:/cockroach/cockroach-data
    networks:
      - crdb_net
    ports:
      - "26257:26257"
      - "8080:8080"

  roach2:
    image: cockroachdb/cockroach:v23.2.0
    hostname: roach2
    command: start --insecure --join=roach1,roach2,roach3 --listen-addr=roach2:26257 --advertise-addr=roach2:26257 --http-addr=roach2:8081
    volumes:
      - roach2_data:/cockroach/cockroach-data
    networks:
      - crdb_net
    ports:
      - "8081:8081"

  roach3:
    image: cockroachdb/cockroach:v23.2.0
    hostname: roach3
    command: start --insecure --join=roach1,roach2,roach3 --listen-addr=roach3:26257 --advertise-addr=roach3:26257 --http-addr=roach3:8082
    volumes:
      - roach3_data:/cockroach/cockroach-data
    networks:
      - crdb_net
    ports:
      - "8082:8082"

volumes:
  roach1_data:
  roach2_data:
  roach3_data:

networks:
  crdb_net:
    driver: bridge
```

## Initializing the Cluster

After the stack starts, run cluster init exactly once:

```bash
docker exec -it $(docker ps -qf name=roach1) \
  ./cockroach init --insecure --host=roach1:26257
```

You should see `Cluster successfully initialized`.

## Creating a Database and User

Connect with the SQL shell and create your application database:

```bash
docker exec -it $(docker ps -qf name=roach1) \
  ./cockroach sql --insecure --host=roach1:26257

-- In the SQL shell:
CREATE DATABASE appdb;
CREATE USER appuser WITH PASSWORD 'apppassword';
GRANT ALL ON DATABASE appdb TO appuser;
```

## Verifying Node Health

Check cluster status from the CockroachDB Admin UI at `http://localhost:8080`, or via the CLI:

```bash
docker exec -it $(docker ps -qf name=roach1) \
  ./cockroach node status --insecure --host=roach1:26257
```

All three nodes should show `is_live: true`.

## Connecting Your Application

CockroachDB is PostgreSQL-wire-compatible. Use your existing PostgreSQL driver:

```python
import psycopg2

conn = psycopg2.connect(
    host="roach1",
    port=26257,
    dbname="appdb",
    user="appuser",
    password="apppassword",
    sslmode="disable"  # for insecure mode; use sslmode=require in production
)
```

## Enabling TLS (Production)

For production, use `--certs-dir` instead of `--insecure` and generate certificates with:

```bash
cockroach cert create-ca --certs-dir=/certs --ca-key=/certs/ca.key
cockroach cert create-node roach1 --certs-dir=/certs --ca-key=/certs/ca.key
cockroach cert create-client root --certs-dir=/certs --ca-key=/certs/ca.key
```

Mount the `/certs` directory as a volume in each service.

## Scaling the Cluster

CockroachDB automatically rebalances data when you add nodes. Add a fourth node to the stack and join it to the existing cluster:

```yaml
  roach4:
    image: cockroachdb/cockroach:v23.2.0
    hostname: roach4
    command: start --insecure --join=roach1,roach2,roach3 --listen-addr=roach4:26257 --advertise-addr=roach4:26257
    volumes:
      - roach4_data:/cockroach/cockroach-data
    networks:
      - crdb_net
```

Data rebalancing begins automatically after the new node joins.
