# How to Troubleshoot HTTP 504 Gateway Timeout Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: HTTP 504, Nginx, Reverse Proxy, Timeout, Debugging, DevOps

Description: Learn how to systematically diagnose and fix HTTP 504 Gateway Timeout errors in reverse proxy setups by identifying slow upstreams and tuning timeout settings.

## What Is a 504 Gateway Timeout?

A `504 Gateway Timeout` means the proxy (Nginx, HAProxy, AWS ALB, etc.) sent a request to the upstream server but did not receive a response within the configured timeout window. Unlike a 502, the upstream is reachable—it's just too slow.

## Common Causes

- Slow database queries holding up the request
- Upstream application processing a large workload
- Upstream overwhelmed under high concurrency
- Network latency between proxy and upstream
- Proxy timeout set too low for legitimate long-running operations
- Deadlocks or stuck threads in the upstream application

## Step 1: Identify Timeout in Nginx Error Log

```bash
# Look for upstream timeout entries
grep "upstream timed out" /var/log/nginx/error.log | tail -20
```

The log entry shows the upstream address, port, and the operation that timed out (read, connect, or send).

## Step 2: Measure Upstream Response Time Directly

```bash
# Time a direct request to the upstream, bypassing the proxy
time curl -s -o /dev/null http://127.0.0.1:3000/slow-endpoint

# Detailed timing
curl -o /dev/null -s -w "Total: %{time_total}s\n" http://127.0.0.1:3000/slow-endpoint
```

If this takes longer than your proxy timeout, you need to either optimize the upstream or increase the timeout.

## Step 3: Tune Nginx Timeout Settings

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        # Time to establish connection with upstream
        proxy_connect_timeout 10s;

        # Time to wait for upstream to send response headers
        proxy_read_timeout 120s;

        # Time to transmit the request to upstream
        proxy_send_timeout 30s;

        # For long-polling or streaming endpoints, set higher
        # proxy_read_timeout 300s;
    }
}
```

Reload after changes: `nginx -t && nginx -s reload`

## Step 4: Profile Slow Database Queries

If your application is backed by a database, enable slow query logging:

```sql
-- MySQL: enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- log queries taking > 1 second
```

```bash
# View slow query log
tail -f /var/log/mysql/slow.log
```

## Step 5: Monitor Upstream Concurrency

```bash
# Count active connections to upstream (e.g., Node.js on port 3000)
ss -tn state established '( dport = :3000 or sport = :3000 )' | wc -l

# Check Nginx upstream status (requires stub_status or upstream_check_module)
curl http://127.0.0.1:8080/nginx_status
```

## Step 6: Add Application-Level Timeouts

Ensure the upstream application itself has timeouts on downstream dependencies:

```python
# Python requests example with a timeout
import requests

# Set connect and read timeout (seconds)
response = requests.get("http://internal-service/api", timeout=(5, 30))
```

## Diagnosing 504 vs 502

| Error | Meaning |
|-------|---------|
| 502   | Upstream unreachable or returned invalid response |
| 504   | Upstream reachable but too slow to respond |

## Conclusion

504 errors indicate a slow upstream, not a broken one. Start by measuring upstream response time directly, then decide whether to optimize the upstream code (preferred) or increase proxy timeouts (acceptable for legitimate long operations). Always add application-level timeouts to prevent cascading slowdowns.
