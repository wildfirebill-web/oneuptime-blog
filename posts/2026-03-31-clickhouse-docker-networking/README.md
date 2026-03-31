# How to Configure ClickHouse Docker Container Networking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Networking, Docker Network, Ports, Container Configuration

Description: Learn how to configure Docker networking for ClickHouse containers including port mappings, custom networks, and inter-container communication.

---

Proper Docker networking configuration ensures ClickHouse is accessible to the right clients while remaining isolated from unintended access. This guide covers port mappings, custom networks, and connecting ClickHouse with other containerized services.

## ClickHouse Default Ports

| Port | Protocol | Purpose |
|---|---|---|
| 8123 | HTTP | REST/HTTP queries |
| 9000 | TCP | Native ClickHouse protocol |
| 9009 | TCP | Inter-server replication |
| 9440 | TCP | Native TLS |
| 8443 | HTTPS | HTTP TLS |

## Basic Port Mapping

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"   # HTTP
      - "9000:9000"   # Native
```

Avoid exposing `9009` to the public network - it is for internal replication only.

## Custom Docker Network

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    networks:
      - analytics

  grafana:
    image: grafana/grafana:latest
    networks:
      - analytics
    environment:
      GF_DATABASE_URL: clickhouse://clickhouse:8123/default

networks:
  analytics:
    driver: bridge
```

Services on the same Docker network communicate using service names as hostnames. Grafana can connect to ClickHouse using `http://clickhouse:8123`.

## Restricting Public Access

To expose ClickHouse only on localhost (not all interfaces):

```yaml
ports:
  - "127.0.0.1:8123:8123"
  - "127.0.0.1:9000:9000"
```

This prevents external access while keeping the service available to other containers on the host.

## Connecting from Another Container

```bash
docker run --rm --network analytics \
  curlimages/curl \
  curl -s 'http://clickhouse:8123/?query=SELECT+version()'
```

## Configuring ClickHouse to Listen on Specific Interfaces

Via a custom config file:

```xml
<!-- config.d/network.xml -->
<clickhouse>
  <listen_host>0.0.0.0</listen_host>
  <tcp_port>9000</tcp_port>
  <http_port>8123</http_port>
</clickhouse>
```

Mount this in Docker Compose:

```yaml
volumes:
  - ./config/network.xml:/etc/clickhouse-server/config.d/network.xml
```

## Testing Connectivity

```bash
# From another container
docker exec my-app-container \
  wget -q -O - 'http://clickhouse:8123/?query=SELECT+1'

# From host
curl -s 'http://localhost:8123/?query=SELECT+version()'
```

## Summary

Configure ClickHouse Docker networking by using custom bridge networks for inter-container communication, binding ports to `127.0.0.1` for local-only access, and never exposing replication port 9009 externally. Service names on the same Docker network resolve automatically as hostnames.
