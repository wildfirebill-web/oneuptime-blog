# How to Monitor RabbitMQ Connections by IPv4 Client Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RabbitMQ, Monitoring, IPv4, Connection, Management API, Rabbitmqctl, Observability

Description: Learn how to monitor active RabbitMQ client connections and filter by IPv4 address using the management API and CLI tools.

---

Monitoring RabbitMQ connections by client IPv4 address helps identify unauthorized connections, diagnose connection leaks, and understand which services are consuming broker resources.

## Method 1: rabbitmqctl

```bash
# List all connections with client addresses

rabbitmqctl list_connections name peer_host peer_port user vhost state channels

# Example output:
# 10.0.0.20:54321 -> 10.0.0.10:5672   appuser   /   running   2
# 10.0.0.21:54322 -> 10.0.0.10:5672   worker1   /   running   1

# Filter by peer (client) IPv4 address
rabbitmqctl list_connections peer_host | grep 10.0.0.20

# Count connections per client IP
rabbitmqctl list_connections peer_host | sort | uniq -c | sort -rn
```

## Method 2: Management HTTP API

The management API returns JSON data for programmatic processing.

```bash
# List all connections (requires management plugin enabled)
curl -s -u admin:password http://10.0.0.10:15672/api/connections | \
  python3 -m json.tool | grep -A2 '"peer_host"'

# Filter for a specific IPv4 address using jq
curl -s -u admin:password http://10.0.0.10:15672/api/connections | \
  jq '[.[] | select(.peer_host == "10.0.0.20")] | length'

# Get full details for connections from a specific IP
curl -s -u admin:password http://10.0.0.10:15672/api/connections | \
  jq '.[] | select(.peer_host == "10.0.0.20") | {name, user, vhost, channels, state}'
```

## Method 3: Management UI

Navigate to `http://10.0.0.10:15672` → **Connections** tab to see all connections with client IPs, usernames, channel counts, and bandwidth statistics.

## Monitoring with the Full Connection Details

```bash
# Get connections with all details: name, IP, user, channels, bytes sent/received
rabbitmqctl list_connections \
  name peer_host peer_port user vhost state channels \
  send_oct recv_oct
```

## Alerting on Too Many Connections from One IP

```bash
#!/bin/bash
# check_rabbitmq_connections.sh
# Alert if any single IPv4 has more than 50 connections

THRESHOLD=50
MGMT_URL="http://10.0.0.10:15672/api/connections"

# Count connections per IP
curl -s -u admin:password "$MGMT_URL" | \
  jq -r '.[].peer_host' | sort | uniq -c | sort -rn | \
  while read count ip; do
    if [ "$count" -gt "$THRESHOLD" ]; then
      echo "ALERT: $ip has $count connections (threshold: $THRESHOLD)"
    fi
  done
```

## Closing Connections from a Specific IP

```bash
# Get connection names for a specific IP
CONN_NAMES=$(curl -s -u admin:password http://10.0.0.10:15672/api/connections | \
  jq -r '.[] | select(.peer_host == "10.0.0.99") | .name')

# Close each connection
while IFS= read -r name; do
  curl -s -u admin:password -X DELETE \
    "http://10.0.0.10:15672/api/connections/$(python3 -c "import urllib.parse; print(urllib.parse.quote('$name', safe=''))")"
done <<< "$CONN_NAMES"
```

## Key Takeaways

- `rabbitmqctl list_connections peer_host` shows client IPv4 addresses for all active connections.
- The management HTTP API (`/api/connections`) provides rich JSON data suitable for monitoring tools.
- Use `jq` to filter and aggregate connection data by client IP programmatically.
- The management UI provides a real-time view of connections including per-connection bandwidth statistics.
