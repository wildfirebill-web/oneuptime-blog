# How to Set Up ClickHouse Dev Environment with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Development Environment, Local Setup, Docker Compose, Developer Tooling

Description: Learn how to set up a complete ClickHouse local development environment with Docker including sample data, monitoring tools, and a web UI for productive development.

---

A good ClickHouse development environment goes beyond just running the server - it includes tools for browsing data, visualizing metrics, and loading sample datasets. This guide builds a full dev stack with Docker Compose.

## Full Dev Stack docker-compose.yml

```yaml
version: '3.8'

services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse-dev
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./config:/etc/clickhouse-server/config.d
      - ./init:/docker-entrypoint-initdb.d
    environment:
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8123/ping"]
      interval: 10s
      retries: 5

  tabix:
    image: spoonest/clickhouse-tabix-web-client:stable
    container_name: clickhouse-ui
    ports:
      - "8080:80"
    depends_on:
      clickhouse:
        condition: service_healthy

  grafana:
    image: grafana/grafana:latest
    container_name: grafana-dev
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    depends_on:
      clickhouse:
        condition: service_healthy

volumes:
  clickhouse_data:
```

## Loading Sample Data

```bash
# Download the ClickHouse airline flights dataset
curl -O https://datasets.clickhouse.com/hits_compatible/hits.csv.gz

# Load into ClickHouse
docker exec -i clickhouse-dev clickhouse-client \
  --query "CREATE TABLE IF NOT EXISTS default.hits (
    WatchID UInt64,
    UserID UInt64,
    URL String,
    EventTime DateTime
  ) ENGINE = MergeTree ORDER BY WatchID"

zcat hits.csv.gz | docker exec -i clickhouse-dev clickhouse-client \
  --query "INSERT INTO default.hits FORMAT CSV"
```

## Useful Development Aliases

```bash
# Add to your ~/.zshrc or ~/.bashrc
alias ch='docker exec -it clickhouse-dev clickhouse-client'
alias ch-query='docker exec clickhouse-dev clickhouse-client --query'

# Usage
ch
ch-query "SELECT version()"
```

## Dev Configuration (config/dev.xml)

```xml
<clickhouse>
  <!-- Disable query limits for development -->
  <profiles>
    <default>
      <max_execution_time>300</max_execution_time>
      <max_memory_usage>8000000000</max_memory_usage>
    </default>
  </profiles>

  <!-- Verbose logging for debugging -->
  <logger>
    <level>debug</level>
  </logger>
</clickhouse>
```

## Quick Start Commands

```bash
# Start the stack
docker compose up -d

# Open ClickHouse client
docker exec -it clickhouse-dev clickhouse-client

# Access Tabix UI
open http://localhost:8080

# Access Grafana
open http://localhost:3000

# Tail logs
docker compose logs -f clickhouse
```

## Summary

A complete ClickHouse dev environment includes the server, a web UI (Tabix), Grafana for dashboards, and sample data for realistic testing. Use shell aliases for quick access, enable verbose logging, and relax query limits for development to avoid hitting production-style constraints during exploration.
