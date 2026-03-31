# How to Set Up ClickHouse Healthchecks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Healthcheck, Monitoring, Kubernetes, Load Balancer, Operations

Description: Learn how to implement ClickHouse healthchecks using the HTTP ping endpoint, SQL probes, and Kubernetes liveness and readiness probes to ensure traffic only reaches healthy nodes.

---

A healthy ClickHouse node must satisfy several conditions: the server process must be running, it must accept connections, queries must execute within acceptable time, and replicated tables must not be in a read-only state. Different layers of your infrastructure require different levels of health verification. This guide covers each layer from raw TCP to Kubernetes probes.

## The Built-in HTTP Ping Endpoint

ClickHouse exposes a lightweight `/ping` endpoint on its HTTP interface (default port 8123) that returns `Ok.` when the server is alive:

```bash
# Simple liveness check
curl -sf http://localhost:8123/ping
# Expected output: Ok.

# With timing
curl -sw "\nTime: %{time_total}s\nHTTP: %{http_code}\n" http://localhost:8123/ping

# Non-zero exit code on failure - useful in scripts
if curl -sf --max-time 5 http://localhost:8123/ping; then
    echo "ClickHouse is alive"
else
    echo "ClickHouse is NOT responding"
    exit 1
fi
```

The `/ping` endpoint does not execute any SQL. It only checks whether the HTTP server thread is alive.

## The /replicas_status Endpoint

For replicated deployments, use the `/replicas_status` endpoint which returns `Ok.` only when all replicated tables have no replication errors:

```bash
curl -sf http://localhost:8123/replicas_status
# Returns "Ok." if all replicas are healthy
# Returns a non-200 status with error details otherwise

# Verbose output
curl -v http://localhost:8123/replicas_status 2>&1 | grep -E "< HTTP|^Ok"
```

## SQL-Based Healthchecks

A deeper probe executes a lightweight SQL query to verify the query engine is functional:

```bash
# Execute a trivial query and check the result
result=$(clickhouse-client --query "SELECT 1" 2>/dev/null)
if [ "$result" = "1" ]; then
    echo "Query engine healthy"
else
    echo "Query engine check FAILED"
    exit 1
fi

# Check via HTTP interface (no client binary needed)
result=$(curl -sf "http://localhost:8123/?query=SELECT+1")
if [ "$result" = "1" ]; then
    echo "HTTP query interface healthy"
fi
```

## Comprehensive Healthcheck Script

Combine multiple checks into a single script suitable for use as a cron job or external probe:

```bash
#!/bin/bash
# /usr/local/bin/clickhouse-healthcheck.sh

set -euo pipefail

HOST="${CLICKHOUSE_HOST:-localhost}"
HTTP_PORT="${CLICKHOUSE_HTTP_PORT:-8123}"
TCP_PORT="${CLICKHOUSE_TCP_PORT:-9000}"
USER="${CLICKHOUSE_USER:-default}"
PASSWORD="${CLICKHOUSE_PASSWORD:-}"
MAX_REPLICATION_LAG="${MAX_REPLICATION_LAG:-120}"
TIMEOUT=5

ERRORS=0

# 1. HTTP ping
if ! curl -sf --max-time "$TIMEOUT" "http://${HOST}:${HTTP_PORT}/ping" > /dev/null; then
    echo "FAIL: HTTP ping failed"
    ERRORS=$((ERRORS + 1))
else
    echo "OK: HTTP ping"
fi

# 2. TCP port reachable
if ! timeout "$TIMEOUT" bash -c "echo > /dev/tcp/${HOST}/${TCP_PORT}" 2>/dev/null; then
    echo "FAIL: TCP port ${TCP_PORT} unreachable"
    ERRORS=$((ERRORS + 1))
else
    echo "OK: TCP port ${TCP_PORT}"
fi

# 3. Query execution
query_result=$(curl -sf --max-time "$TIMEOUT" \
    "http://${HOST}:${HTTP_PORT}/?user=${USER}&password=${PASSWORD}&query=SELECT+1" 2>/dev/null || true)
if [ "$query_result" = "1" ]; then
    echo "OK: Query execution"
else
    echo "FAIL: Query execution returned '${query_result}'"
    ERRORS=$((ERRORS + 1))
fi

# 4. Replication lag
lag=$(curl -sf --max-time "$TIMEOUT" \
    "http://${HOST}:${HTTP_PORT}/?user=${USER}&password=${PASSWORD}&query=SELECT+max(absolute_delay)+FROM+system.replicas" \
    2>/dev/null || echo "999999")
if [ "$lag" -le "$MAX_REPLICATION_LAG" ] 2>/dev/null; then
    echo "OK: Replication lag ${lag}s"
else
    echo "WARN: Replication lag ${lag}s exceeds threshold ${MAX_REPLICATION_LAG}s"
fi

# 5. Readonly replicas
readonly_count=$(curl -sf --max-time "$TIMEOUT" \
    "http://${HOST}:${HTTP_PORT}/?user=${USER}&password=${PASSWORD}&query=SELECT+countIf(is_readonly)+FROM+system.replicas" \
    2>/dev/null || echo "999")
if [ "$readonly_count" = "0" ]; then
    echo "OK: No readonly replicas"
else
    echo "FAIL: ${readonly_count} replica(s) are readonly"
    ERRORS=$((ERRORS + 1))
fi

# Summary
if [ "$ERRORS" -eq 0 ]; then
    echo "OVERALL: HEALTHY"
    exit 0
else
    echo "OVERALL: UNHEALTHY (${ERRORS} check(s) failed)"
    exit 1
fi
```

Make it executable and test it:

```bash
chmod +x /usr/local/bin/clickhouse-healthcheck.sh
/usr/local/bin/clickhouse-healthcheck.sh
```

## Kubernetes Liveness and Readiness Probes

Kubernetes uses two separate probes:

- **Liveness probe**: kills and restarts the container if the probe fails
- **Readiness probe**: removes the pod from service endpoints so no traffic is sent to it

```yaml
# clickhouse-deployment.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: clickhouse
spec:
  template:
    spec:
      containers:
        - name: clickhouse
          image: clickhouse/clickhouse-server:24.3
          ports:
            - containerPort: 8123
              name: http
            - containerPort: 9000
              name: tcp
          livenessProbe:
            httpGet:
              path: /ping
              port: 8123
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /replicas_status
              port: 8123
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 2
          startupProbe:
            httpGet:
              path: /ping
              port: 8123
            failureThreshold: 30
            periodSeconds: 10
```

For more sophisticated readiness logic (such as checking replication lag), use an exec probe:

```yaml
readinessProbe:
  exec:
    command:
      - /bin/bash
      - -c
      - |
        curl -sf http://localhost:8123/ping &&
        lag=$(curl -sf "http://localhost:8123/?query=SELECT+max(absolute_delay)+FROM+system.replicas") &&
        [ "${lag:-0}" -le 60 ]
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

## HAProxy Healthcheck Configuration

For load-balanced deployments behind HAProxy:

```text
# haproxy.cfg
backend clickhouse_http
    balance roundrobin
    option httpchk GET /ping
    http-check expect string Ok.
    server ch-node-1 clickhouse-1:8123 check inter 5s fall 3 rise 2
    server ch-node-2 clickhouse-2:8123 check inter 5s fall 3 rise 2
    server ch-node-3 clickhouse-3:8123 check inter 5s fall 3 rise 2

backend clickhouse_tcp
    balance leastconn
    option tcp-check
    tcp-check connect port 9000
    server ch-node-1 clickhouse-1:9000 check inter 5s fall 3 rise 2
    server ch-node-2 clickhouse-2:9000 check inter 5s fall 3 rise 2
    server ch-node-3 clickhouse-3:9000 check inter 5s fall 3 rise 2
```

## Monitoring Healthcheck Results

Log healthcheck history to a dedicated table for trend analysis:

```sql
CREATE TABLE healthcheck_log
(
    check_time   DateTime DEFAULT now(),
    host         String,
    check_name   String,
    status       Enum8('ok' = 1, 'fail' = 0, 'warn' = 2),
    value        Float64,
    message      String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(check_time)
ORDER BY (host, check_name, check_time)
TTL check_time + INTERVAL 90 DAY;
```

## Summary

ClickHouse healthchecks should be layered: use `/ping` for liveness (is the process alive?), `/replicas_status` for replication-aware readiness (is this node suitable for traffic?), and SQL probes for deeper verification of query engine availability and replication lag. In Kubernetes, map liveness to `/ping` and readiness to `/replicas_status` so that nodes recovering from replication issues are drained from the load balancer automatically rather than serving stale data.
