# How to Configure Elasticsearch to Bind to IPv6 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Elasticsearch, Database, network.host, Search Engine

Description: Learn how to configure Elasticsearch to bind to IPv6 addresses for HTTP and transport communication, enabling IPv6-only and dual-stack deployments.

## Elasticsearch Network Configuration

```yaml
# /etc/elasticsearch/elasticsearch.yml

# Bind to all interfaces (IPv4 and IPv6)
network.host: 0.0.0.0

# Bind to specific IPv6 address
network.host: "2001:db8::10"

# Bind to IPv6 loopback only (local testing)
network.host: "::1"

# Separate HTTP and transport bind addresses
network.bind_host: "2001:db8::10"
network.publish_host: "2001:db8::10"

# Or using array for multiple addresses
network.bind_host: ["::1", "2001:db8::10"]
```

## Special Values for network.host

```yaml
# Elasticsearch provides special values:
# _local_    = loopback addresses (127.0.0.1, ::1)
# _site_     = site-local addresses
# _global_   = global addresses (public)
# _[iface]_  = addresses of a specific interface (e.g., _eth0_)
# 0.0.0.0    = all IPv4
# ::         = all IPv6
# ::0        = all IPv6 (alternative notation)

# IPv6-specific binding using _global_:
network.host: ["_local_", "_global_"]
```

## Full Configuration Example

```yaml
# /etc/elasticsearch/elasticsearch.yml

cluster.name: my-cluster
node.name: node-1

# Network
network.host: "2001:db8::10"
http.port: 9200
transport.port: 9300

# Cluster discovery with IPv6
discovery.seed_hosts:
  - "[2001:db8::10]:9300"
  - "[2001:db8::11]:9300"
  - "[2001:db8::12]:9300"

cluster.initial_master_nodes:
  - node-1
  - node-2
  - node-3

# Security (Elasticsearch 8+)
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

## Apply Configuration

```bash
# Restart Elasticsearch
systemctl restart elasticsearch

# Verify listening on IPv6
ss -6 -tlnp | grep java
# Should show [::]:9200 (HTTP) and [::]:9300 (transport)

# Test HTTP API over IPv6
curl -6 http://[2001:db8::10]:9200/

# Test with authentication (Elasticsearch 8+)
curl -6 -u elastic:password https://[2001:db8::10]:9200/
```

## Test Elasticsearch IPv6 Operations

```bash
# Check cluster health
curl -6 http://[2001:db8::10]:9200/_cluster/health?pretty

# Create an index
curl -6 -X PUT http://[2001:db8::10]:9200/myindex

# Index a document
curl -6 -X POST http://[2001:db8::10]:9200/myindex/_doc/1 \
    -H "Content-Type: application/json" \
    -d '{"title": "IPv6 Test Document"}'

# Search
curl -6 http://[2001:db8::10]:9200/myindex/_search?pretty
```

## JVM IPv6 Preferences

```bash
# Elasticsearch runs on JVM - configure JVM for IPv6

# /etc/elasticsearch/jvm.options.d/ipv6.options
cat > /etc/elasticsearch/jvm.options.d/ipv6.options << 'EOF'
# Prefer IPv6 stack
-Djava.net.preferIPv6Addresses=true

# Or prefer IPv4 (default JVM behavior)
# -Djava.net.preferIPv4Stack=true
EOF
```

## Summary

Configure Elasticsearch IPv6 with `network.host: "2001:db8::10"` or `network.host: ["::1", "2001:db8::10"]` in `elasticsearch.yml`. For cluster nodes, set `discovery.seed_hosts` with IPv6 addresses in brackets. Restart with `systemctl restart elasticsearch`. For JVM-level IPv6 preferences, add `-Djava.net.preferIPv6Addresses=true` to JVM options. Test with `curl -6 http://[2001:db8::10]:9200/_cluster/health`.
