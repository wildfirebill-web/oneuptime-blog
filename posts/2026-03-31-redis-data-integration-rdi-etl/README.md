# How to Use Redis Data Integration (RDI) for ETL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ETL, Data Integration, Pipeline, Database

Description: Learn how Redis Data Integration (RDI) captures database changes via CDC and writes them to Redis in real time, enabling low-latency data pipelines.

---

Redis Data Integration (RDI) is a product from Redis Ltd. that uses Change Data Capture (CDC) to continuously sync data from relational databases into Redis. It eliminates the need to write custom ETL code by providing a declarative pipeline configuration.

## What RDI Does

RDI sits between your source database (PostgreSQL, MySQL, Oracle, SQL Server) and Redis. It:

1. Uses Debezium under the hood to capture row-level changes
2. Transforms records using a YAML-based config
3. Writes data into Redis as hashes, JSON, streams, or sets

## Installing RDI

RDI runs as a set of Docker containers (or on Kubernetes). Install with:

```bash
# Download the RDI installer
wget https://github.com/RedisLabs/redis-di/releases/latest/download/redis-di-linux.tar.gz
tar xf redis-di-linux.tar.gz

# Install RDI on a Redis Enterprise target
redis-di create --cluster-host <CLUSTER_HOST> --cluster-port 9443 \
  --cluster-user <USER> --cluster-password <PASS> \
  --rdi-port 13000
```

## Configuring a Source Database

Create a `config.yaml` to define your source:

```yaml
sources:
  mysql:
    type: cdc
    logging:
      level: info
    connection:
      type: mysql
      host: mysql-host
      port: 3306
      database: orders_db
      user: debezium
      password: secret
```

Grant Debezium the necessary MySQL privileges:

```sql
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT
ON *.* TO 'debezium'@'%';
```

## Writing Transformation Jobs

Each pipeline job maps a database table to a Redis data structure. Create `jobs/orders.yaml`:

```yaml
source:
  server-name: mysql-host
  db: orders_db
  table: orders

transform:
  - uses: add_field
    with:
      field: full_name
      language: jmespath
      expression: "concat([customer_first, ' ', customer_last])"

output:
  - uses: redis.write
    with:
      data_type: hash
      key:
        expression: concat(['order:', id])
        language: jmespath
```

## Deploying the Pipeline

```bash
# Validate config
redis-di scaffold --db-type mysql

# Deploy jobs
redis-di deploy --dir ./jobs/

# Check status
redis-di status
```

## Verifying Data in Redis

After the pipeline runs, confirm data is flowing:

```bash
redis-cli HGETALL "order:12345"
# 1) "id"
# 2) "12345"
# 3) "full_name"
# 4) "Jane Doe"
# 5) "total"
# 6) "99.99"
```

## Common Transformation Patterns

```yaml
# Filter only active records
transform:
  - uses: filter
    with:
      expression: status == 'active'
      language: jmespath

# Rename fields
  - uses: rename_field
    with:
      from_field: cust_id
      to_field: customer_id
```

## Monitoring RDI

```bash
# Check pipeline metrics
redis-di status

# View ingestion lag
redis-di get-rejected --shard 1
```

## Summary

Redis Data Integration simplifies ETL from relational databases to Redis using CDC and declarative YAML jobs. It removes the need for custom Debezium consumers and provides built-in transformation primitives. RDI is best suited for teams running Redis Enterprise who need low-latency database mirroring.
