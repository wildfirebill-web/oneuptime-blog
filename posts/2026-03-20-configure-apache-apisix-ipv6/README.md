# How to Configure Apache APISIX for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: APISIX, API Gateway, IPv6, Networking, Lua, Nginx

Description: Enable IPv6 support in Apache APISIX by configuring NGINX listeners, Admin API bindings, and upstream service definitions for dual-stack operation.

## Introduction

Apache APISIX is built on top of NGINX and OpenResty, so enabling IPv6 involves configuring NGINX listener directives via APISIX's `config.yaml`. APISIX can proxy both IPv4 and IPv6 traffic simultaneously with minor configuration changes.

## Prerequisites

- Apache APISIX 3.x installed
- etcd cluster for configuration storage
- An IPv6-enabled host

## Step 1: Update config.yaml for IPv6 Listeners

APISIX's main configuration file controls NGINX listener bindings.

```yaml
# /usr/local/apisix/conf/config.yaml

apisix:
  # Node identification
  node_listen:
    # Listen on IPv4 wildcard
    - port: 9080
      ip: 0.0.0.0
    # Listen on IPv6 wildcard
    - port: 9080
      ip: "::"
  # SSL listener on both stacks
  ssl:
    enable: true
    listen:
      - port: 9443
        ip: 0.0.0.0
      - port: 9443
        ip: "::"

admin_api:
  admin_listen:
    ip: "0.0.0.0"
    port: 9180
  admin_listen_ipv6:
    ip: "::"
    port: 9180

etcd:
  hosts:
    - "http://[::1]:2379"
  prefix: /apisix
```

## Step 2: Define an Upstream with IPv6 Nodes

Use the Admin API to create an upstream that includes IPv6 backend addresses.

```bash
# Create an upstream with IPv6 backend nodes

curl -X PUT http://[::1]:9180/apisix/admin/upstreams/1 \
  -H "X-API-KEY: your-admin-key" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "roundrobin",
    "nodes": {
      "[2001:db8::10]:8080": 1,
      "[2001:db8::11]:8080": 1
    },
    "checks": {
      "active": {
        "http_path": "/health",
        "healthy": {
          "interval": 2,
          "successes": 1
        },
        "unhealthy": {
          "interval": 1,
          "http_failures": 2
        }
      }
    }
  }'
```

## Step 3: Create a Route

```bash
# Create a route that uses the IPv6 upstream
curl -X PUT http://[::1]:9180/apisix/admin/routes/1 \
  -H "X-API-KEY: your-admin-key" \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "/api/*",
    "name": "ipv6-api-route",
    "upstream_id": "1",
    "plugins": {
      "proxy-rewrite": {
        "regex_uri": ["/api/(.*)", "/$1"]
      }
    }
  }'
```

## Step 4: Verify Listeners and Test

```bash
# Check APISIX is bound to IPv6
ss -tlnp | grep -E "9080|9443"

# Test the proxy over IPv6
curl -6 http://[::1]:9080/api/health

# Test via hostname resolving to IPv6
curl -6 http://mygateway.example.com/api/health

# Check Admin API over IPv6
curl -6 http://[::1]:9180/apisix/admin/routes \
  -H "X-API-KEY: your-admin-key"
```

## Step 5: Enable the IP Restriction Plugin for IPv6 Subnets

```bash
# Add IP restriction to a route for IPv6 CIDR blocks
curl -X PATCH http://[::1]:9180/apisix/admin/routes/1 \
  -H "X-API-KEY: your-admin-key" \
  -H "Content-Type: application/json" \
  -d '{
    "plugins": {
      "ip-restriction": {
        "whitelist": [
          "2001:db8::/32",
          "192.168.0.0/24"
        ]
      }
    }
  }'
```

## Common Issues

- **NGINX not binding to IPv6**: Verify the `node_listen` entries use `ip: "::"` not `ip: "::0"`.
- **etcd connection refused**: Ensure etcd is listening on `::1` if using loopback.
- **Upstream health checks failing**: Health check connections inherit the upstream address family - no extra config needed.

## Conclusion

APISIX's YAML-based configuration cleanly separates listener definitions per IP family. Adding `ip: "::"` entries alongside IPv4 ones enables dual-stack with minimal risk. Use OneUptime's HTTP monitors to probe both IPv4 and IPv6 paths of your APISIX gateway continuously.
