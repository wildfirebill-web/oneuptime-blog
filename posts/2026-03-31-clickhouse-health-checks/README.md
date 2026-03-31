# How to Set Up ClickHouse Health Checks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Health Check, Monitoring, Kubernetes, Operations

Description: Learn how to configure ClickHouse health check endpoints, integrate with load balancers and Kubernetes, and build custom liveness probes.

---

ClickHouse ships with built-in HTTP endpoints for health checking. Setting these up correctly ensures your load balancers route traffic only to healthy nodes and your orchestration platform restarts unhealthy pods automatically.

## Built-In HTTP Health Endpoints

ClickHouse exposes several health check endpoints on the HTTP port (default 8123):

```bash
# Simple ping - returns "Ok." if server is running
curl http://localhost:8123/ping

# Replicas check - returns "Ok." only if no replica lag
curl http://localhost:8123/replicas_status

# Custom health query via query parameter
curl "http://localhost:8123/?query=SELECT+1"
```

The `/ping` endpoint returns HTTP 200 with body `Ok.` when the server is alive. The `/replicas_status` endpoint returns 200 only when all replicated tables are caught up.

## Configuring Custom Health Check Queries

For more thorough checks, use the query endpoint to run SQL and verify results:

```bash
# Check that the server can execute queries and has an expected table
curl "http://localhost:8123/?query=SELECT+count()+FROM+system.tables+FORMAT+JSON"
```

## Kubernetes Liveness and Readiness Probes

Configure ClickHouse pods with appropriate probes in your deployment manifest:

```yaml
livenessProbe:
  httpGet:
    path: /ping
    port: 8123
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /replicas_status
    port: 8123
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

Use `/ping` for liveness (is the process alive?) and `/replicas_status` for readiness (is the node ready to serve traffic?). This prevents routing queries to nodes that are still catching up on replication.

## Load Balancer Health Checks

For HAProxy:

```text
backend clickhouse_backend
    option httpchk GET /ping HTTP/1.1\r\nHost:\ clickhouse
    server ch1 10.0.0.1:8123 check inter 5s rise 2 fall 3
    server ch2 10.0.0.2:8123 check inter 5s rise 2 fall 3
```

For Nginx upstream health checks:

```nginx
upstream clickhouse {
    server 10.0.0.1:8123;
    server 10.0.0.2:8123;
    check interval=5000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "GET /ping HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx;
}
```

## Custom Health Check Script

For monitoring systems that require a script-based check:

```bash
#!/bin/bash
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8123/ping)
if [ "$RESPONSE" -eq 200 ]; then
    echo "ClickHouse is healthy"
    exit 0
else
    echo "ClickHouse health check failed: HTTP $RESPONSE"
    exit 1
fi
```

## Checking Replication Health via SQL

For a deeper replication health check, query the system tables directly:

```sql
SELECT
    database,
    table,
    is_leader,
    inserts_in_queue,
    merges_in_queue,
    queue_oldest_time,
    log_max_index - log_pointer AS replication_lag
FROM system.replicas
WHERE is_session_expired = 1 OR inserts_in_queue > 100 OR merges_in_queue > 50;
```

## Summary

ClickHouse's built-in `/ping` and `/replicas_status` endpoints cover most health check needs. Use `/ping` for liveness probes and `/replicas_status` for readiness probes in Kubernetes. For load balancers, configure HTTP checks against `/ping` with appropriate rise/fall thresholds to handle brief restarts gracefully.
