# How to Configure SQLite Name Resolution in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, SQLite, Name Resolution, Self-Hosted, Service Discovery

Description: Learn how to configure Dapr's SQLite name resolution component for reliable service discovery in self-hosted environments where mDNS is unavailable.

---

## Why Use SQLite for Name Resolution?

Dapr's default self-hosted name resolution uses mDNS, which relies on multicast networking. In many environments - Docker containers, cloud VMs, or systems with restrictive firewalls - multicast is unavailable. SQLite name resolution solves this by storing service registrations in a local SQLite database file accessible to all Dapr sidecars on the same host.

This is ideal for development environments, single-host deployments, and CI/CD pipelines.

## Configuring the SQLite Name Resolution Component

Create a component file for SQLite name resolution:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: nameresolution
spec:
  type: nameresolution.sqlite
  version: v1
  metadata:
    - name: connectionString
      value: "/tmp/dapr-nameresolution.db"
    - name: timeout
      value: "10s"
    - name: cleanupInterval
      value: "1m"
    - name: updateInterval
      value: "5s"
```

Place this file in your components directory:

```bash
mkdir -p ~/.dapr/components
cp nameresolution-sqlite.yaml ~/.dapr/components/
```

## Running Multiple Services with SQLite Resolution

All services on the same host share the same SQLite database file. Start multiple services pointing to the same components directory:

```bash
# Terminal 1
dapr run --app-id order-service \
  --app-port 8080 \
  --components-path ~/.dapr/components \
  -- ./order-service

# Terminal 2
dapr run --app-id payment-service \
  --app-port 8081 \
  --components-path ~/.dapr/components \
  -- ./payment-service
```

Both services register themselves in the shared SQLite database and can discover each other.

## Verifying Service Registration

Inspect the SQLite database to confirm services are registered:

```bash
sqlite3 /tmp/dapr-nameresolution.db \
  "SELECT appID, address, port, updateTime FROM hosts;"
```

Expected output:

```
order-service|127.0.0.1|50001|2026-03-31T12:00:00Z
payment-service|127.0.0.1|50002|2026-03-31T12:00:01Z
```

## Using SQLite in Docker Compose

Mount a shared volume so all containers use the same database:

```yaml
version: "3.8"
services:
  order-service:
    image: myapp/order-service
    volumes:
      - dapr-db:/tmp
      - ./components:/components
    environment:
      - DAPR_COMPONENTS_PATH=/components

  payment-service:
    image: myapp/payment-service
    volumes:
      - dapr-db:/tmp
      - ./components:/components

volumes:
  dapr-db:
```

## Configuration Parameters Explained

| Parameter | Description | Default |
|-----------|-------------|---------|
| `connectionString` | Path to the SQLite database file | Required |
| `timeout` | Database operation timeout | `10s` |
| `cleanupInterval` | How often stale entries are removed | `1m` |
| `updateInterval` | How often the host entry is refreshed | `5s` |

Tune `updateInterval` and `cleanupInterval` based on your deployment. For fast-cycling containers, use shorter intervals to ensure stale entries are cleared quickly:

```yaml
metadata:
  - name: updateInterval
    value: "2s"
  - name: cleanupInterval
    value: "10s"
```

## Limitations

SQLite name resolution is limited to services that share access to the database file. It is not suitable for distributed, multi-host environments. For multi-host setups, use Consul or Kubernetes DNS instead.

## Summary

Dapr's SQLite name resolution component stores service registrations in a local SQLite file, making it a reliable alternative to mDNS in Docker or containerized environments. Configure a shared volume so all services access the same database, and tune the update and cleanup intervals for your deployment's needs.
