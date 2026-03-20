# How to Troubleshoot Traefik Routing Issues with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Traefik, Troubleshooting, Routing, Debugging

Description: Learn how to diagnose and fix common Traefik routing problems when running containers via Portainer, from 404 errors to certificate failures and misconfigured labels.

## Common Traefik Routing Problems

| Symptom | Common Cause |
|---------|-------------|
| `404 page not found` | Router not created, missing `traefik.enable=true` |
| `502 Bad Gateway` | Wrong port, container not on proxy network |
| Certificate not issued | Port 80 blocked, wrong domain |
| Redirect loop | Redirect on both HTTP and HTTPS |
| Dashboard 401 | Wrong auth hash format |

## Step 1: Enable Debug Logging

```yaml
# traefik.yml — temporarily enable debug logging
log:
  level: DEBUG

accessLog: {}
```

```bash
# Tail Traefik logs
docker logs traefik --follow 2>&1

# Filter for specific domain
docker logs traefik 2>&1 | grep "yourdomain.com"
```

## Step 2: Check Router Discovery

```bash
# List all discovered routers via Traefik API
curl http://localhost:8080/api/http/routers 2>/dev/null | jq '.[] | {name, rule, status}'

# A healthy router shows:
# {
#   "name": "portainer@docker",
#   "rule": "Host(`portainer.example.com`)",
#   "status": "enabled"
# }
```

If the router is missing entirely, `traefik.enable=true` is likely absent or the container isn't on the proxy network.

## Step 3: Check Service Configuration

```bash
# List services
curl http://localhost:8080/api/http/services | jq '.[] | {name, type}'

# Check if Portainer service resolves to correct backend
curl http://localhost:8080/api/http/services/portainer@docker | jq '.loadBalancer.servers'
# Should show: [{"url": "http://172.20.0.x:9000"}]
```

## Step 4: Verify Container is on Proxy Network

```bash
# Check container networks
docker inspect portainer | jq '.[].NetworkSettings.Networks | keys'
# Must include "proxy"

# If missing, connect it
docker network connect proxy portainer
```

## Step 5: Fix 502 Bad Gateway

502 means Traefik can reach the router but not the backend:

```bash
# Check the service port label
docker inspect portainer | jq '.[].Config.Labels | to_entries[] | select(.key | contains("loadbalancer.server.port"))'

# Common mistake: using HTTPS port 9443 instead of 9000
# Wrong:
# - "traefik.http.services.portainer.loadbalancer.server.port=9443"
# Right:
# - "traefik.http.services.portainer.loadbalancer.server.port=9000"
```

For the Portainer HTTPS interface, you need to set the scheme:

```yaml
- "traefik.http.services.portainer.loadbalancer.server.port=9443"
- "traefik.http.services.portainer.loadbalancer.server.scheme=https"
- "traefik.http.services.portainer.loadbalancer.serversTransport=insecureTransport"
```

## Step 6: Fix Label Syntax Errors

Labels in Docker Compose must escape `$` signs:

```yaml
# Wrong — shell will interpret $apr1
- "traefik.http.middlewares.auth.basicauth.users=admin:$apr1$hash"

# Correct — double $ escapes in YAML/Docker Compose
- "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$hash"
```

## Step 7: Check Certificate Issues

```bash
# Check acme.json for your domain
cat /opt/traefik/data/acme.json | jq '.letsencrypt.Certificates[] | select(.domain.main == "portainer.example.com")'

# Check if port 80 is accessible
curl -I http://portainer.example.com

# Check Traefik ACME logs
docker logs traefik 2>&1 | grep -i "acme\|certificate\|error"
```

## Diagnostic Script

```bash
#!/bin/bash
DOMAIN="${1:-portainer.example.com}"

echo "=== Traefik Routing Diagnostics for $DOMAIN ==="

echo -e "\n1. Container running?"
docker ps | grep portainer

echo -e "\n2. Container labels:"
docker inspect portainer | jq '.[].Config.Labels | to_entries[] | select(.key | startswith("traefik"))'

echo -e "\n3. Traefik router for domain:"
curl -s http://localhost:8080/api/http/routers | jq ".[] | select(.rule | contains(\"$DOMAIN\"))"

echo -e "\n4. Service health:"
curl -s http://localhost:8080/api/http/services | jq ".[] | select(.name | contains(\"portainer\"))"

echo -e "\n5. Certificate status:"
cat /opt/traefik/data/acme.json | jq ".letsencrypt.Certificates[] | select(.domain.main == \"$DOMAIN\") | .domain"
```

## Conclusion

Most Traefik routing issues with Portainer-deployed containers fall into three categories: missing `traefik.enable=true`, wrong network configuration, or incorrect port labels. Using the Traefik API and debug logging together, these are usually diagnosable within minutes.
