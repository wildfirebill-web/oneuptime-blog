# How to Deploy TimescaleDB via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, TimescaleDB, PostgreSQL, Time Series, Database, Metrics

Description: Learn how to deploy TimescaleDB via Portainer for time-series workloads using PostgreSQL-compatible hypertables and automatic data compression.

---

TimescaleDB is a PostgreSQL extension that adds hypertables, automatic partitioning, and data compression for time-series workloads. It's fully SQL-compatible, making it a drop-in replacement for PostgreSQL in time-series applications.

## Stack Definition

```yaml
version: "3.8"

services:
  timescaledb:
    image: timescale/timescaledb:latest-pg15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgrespassword
      POSTGRES_DB: metrics
    ports:
      - "5432:5432"
    volumes:
      - timescale_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - tsdb_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d metrics"]
      interval: 10s
      timeout: 5s
      retries: 5

  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: adminpassword
    ports:
      - "5050:80"
    networks:
      - tsdb_net
    depends_on:
      timescaledb:
        condition: service_healthy

volumes:
  timescale_data:

networks:
  tsdb_net:
    driver: bridge
```

## Initialization SQL

Create `init.sql` to enable TimescaleDB and create a hypertable:

```sql
-- Enable the TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Create a metrics table
CREATE TABLE IF NOT EXISTS sensor_readings (
    time        TIMESTAMPTZ       NOT NULL,
    sensor_id   TEXT              NOT NULL,
    location    TEXT              NOT NULL,
    temperature DOUBLE PRECISION  NULL,
    humidity    DOUBLE PRECISION  NULL
);

-- Convert to a hypertable (automatically partitioned by time)
SELECT create_hypertable('sensor_readings', 'time');

-- Create an index for fast queries by sensor
CREATE INDEX ON sensor_readings (sensor_id, time DESC);
```

## Inserting and Querying Time-Series Data

Use standard SQL to interact with hypertables:

```sql
-- Insert sensor readings
INSERT INTO sensor_readings (time, sensor_id, location, temperature, humidity)
VALUES
    (NOW(), 'sensor-001', 'server-room', 22.5, 45.2),
    (NOW() - INTERVAL '5 minutes', 'sensor-001', 'server-room', 23.1, 44.8);

-- Query the last hour of readings with 5-minute averages
SELECT
    time_bucket('5 minutes', time) AS bucket,
    sensor_id,
    AVG(temperature) AS avg_temp,
    AVG(humidity)    AS avg_humidity
FROM sensor_readings
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY bucket, sensor_id
ORDER BY bucket DESC;
```

## Enabling Automatic Compression

Compress chunks older than 7 days to reduce storage:

```sql
-- Enable compression on the hypertable
ALTER TABLE sensor_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id'
);

-- Set a compression policy (runs automatically)
SELECT add_compression_policy('sensor_readings', INTERVAL '7 days');
```

## Data Retention Policy

Automatically drop data older than 90 days:

```sql
SELECT add_retention_policy('sensor_readings', INTERVAL '90 days');
```

## Connecting a Python Application

Use `psycopg2` or `asyncpg` since TimescaleDB is PostgreSQL-compatible:

```python
import psycopg2
from datetime import datetime

conn = psycopg2.connect(
    host="timescaledb",
    port=5432,
    dbname="metrics",
    user="postgres",
    password="postgrespassword"
)

cur = conn.cursor()
cur.execute(
    "INSERT INTO sensor_readings (time, sensor_id, location, temperature) VALUES (%s, %s, %s, %s)",
    (datetime.utcnow(), "sensor-001", "server-room", 22.5)
)
conn.commit()
```

## Grafana Integration

TimescaleDB works with the standard Grafana PostgreSQL data source. Add it with:

- Host: `timescaledb:5432`
- Database: `metrics`
- User/Password: as configured
- TLS mode: disable (for internal Docker network)

Use time-series panels with `$__timeFilter(time)` macros for automatic time range filtering.
