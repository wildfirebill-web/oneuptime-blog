# How to Set Up ClickHouse with Docker Desktop

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Docker Desktop, macOS, Windows, Local Development

Description: Learn how to run ClickHouse locally using Docker Desktop on macOS or Windows, connect with client tools, and manage your instance through the Docker Desktop UI.

---

Docker Desktop is the easiest way to run ClickHouse locally on macOS or Windows without installing ClickHouse natively. This guide walks through setup, connection, and common operations using Docker Desktop.

## Prerequisites

- Docker Desktop installed on macOS or Windows
- Docker Desktop running (check system tray icon)

## Quickstart with Docker Run

The fastest way to start:

```bash
docker run -d \
  --name clickhouse-local \
  -p 8123:8123 \
  -p 9000:9000 \
  -e CLICKHOUSE_PASSWORD=password \
  -v clickhouse-data:/var/lib/clickhouse \
  clickhouse/clickhouse-server:latest
```

## Using Docker Compose (Recommended)

Create a `docker-compose.yml` in your project directory:

```yaml
version: '3.8'

services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse-local
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse_data:/var/lib/clickhouse
    environment:
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1
      CLICKHOUSE_PASSWORD: password

volumes:
  clickhouse_data:
```

Start with:

```bash
docker compose up -d
```

## Managing via Docker Desktop UI

In the Docker Desktop application:
1. Click "Containers" in the left sidebar
2. Find `clickhouse-local`
3. Click the container to view logs, inspect ports, and access the terminal

From the terminal icon, you can open a shell directly:

```bash
clickhouse-client --password password
```

## Connecting from Your Host

Using `clickhouse-client` (install via Homebrew on Mac):

```bash
brew install clickhouse
clickhouse client --host localhost --password password
```

Using HTTP with curl:

```bash
curl -u default:password 'http://localhost:8123/?query=SELECT+version()'
```

## Resource Limits on Docker Desktop

Docker Desktop on macOS/Windows runs in a VM. By default, it may have limited memory. Increase resources:

1. Docker Desktop - Settings - Resources
2. Set Memory to at least 4 GB for comfortable ClickHouse usage
3. Set CPUs to 2+

## Checking ClickHouse Health

```bash
curl -s http://localhost:8123/ping
# Expected: Ok.
```

## Stopping and Restarting

```bash
# Stop
docker compose stop

# Start again (data persists in named volume)
docker compose start

# Full teardown
docker compose down
```

## Summary

Docker Desktop provides a GUI-friendly way to run ClickHouse locally on macOS or Windows. Use Docker Compose for reproducible setup, allocate sufficient memory in Docker Desktop resource settings, and connect via localhost on the standard ClickHouse ports. Named volumes ensure your data survives container restarts.
