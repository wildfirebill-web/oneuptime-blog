# How to Configure Custom Host File Entries for Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Networking, Hosts File, DNS, Container Configuration

Description: Add custom /etc/hosts entries to Docker containers using extra_hosts for hostname-to-IP mappings without requiring a DNS server, managed through Portainer.

## Introduction

The `extra_hosts` directive in Docker Compose adds entries directly to a container's `/etc/hosts` file. This is useful when you need to map hostnames to specific IPs without deploying a DNS server - useful for legacy application integration, testing against mock services, or overriding DNS for specific hostnames. This guide covers all patterns for custom host entries in Portainer-managed containers.

## Step 1: Basic extra_hosts Configuration

```yaml
# docker-compose.yml - Custom /etc/hosts entries

version: "3.8"

services:
  api:
    image: myapp/api:latest
    extra_hosts:
      # Map hostname to specific IP
      - "database.internal:192.168.1.100"
      - "cache.internal:192.168.1.101"
      - "legacy-api.corp:10.0.0.50"

      # Override external hostnames for testing
      - "payment-gateway.com:192.168.1.200"  # Point to mock service

      # Use host-gateway to reach Docker host
      - "host.docker.internal:host-gateway"
```

```bash
# Verify the entries were added
docker exec api_container cat /etc/hosts
# Shows:
# 192.168.1.100  database.internal
# 192.168.1.101  cache.internal
# 10.0.0.50      legacy-api.corp
# 192.168.1.200  payment-gateway.com
```

## Step 2: Access the Docker Host from Containers

A common use case is connecting from a container to services running on the Docker host itself:

```yaml
version: "3.8"

services:
  app:
    image: myapp:latest
    extra_hosts:
      # host-gateway is a special value resolved to host's IP
      - "host.docker.internal:host-gateway"
    environment:
      # Now this works: connect to host's PostgreSQL
      - DB_HOST=host.docker.internal
      - DB_PORT=5432
```

```bash
# Or add manually when running containers
docker run --add-host host.docker.internal:host-gateway myapp:latest

# Verify from inside container
docker exec app_container ping host.docker.internal
# Returns: PING host.docker.internal (172.17.0.1)
```

## Step 3: Multiple Aliases for Same IP

```yaml
version: "3.8"

services:
  worker:
    image: myapp/worker:latest
    extra_hosts:
      # Primary hostname
      - "db.internal:10.0.1.10"
      # Alias (same IP)
      - "postgres.internal:10.0.1.10"
      # Legacy name (same IP)
      - "old-db.corp:10.0.1.10"

      # All resolve to the same server
      # Application can use any of these names
```

## Step 4: Override DNS for Development/Testing

```yaml
# Development override: point production domains to local mocks
version: "3.8"

services:
  api_test:
    image: myapp/api:latest
    extra_hosts:
      # Redirect external API calls to local mock servers
      - "api.stripe.com:127.0.0.1"       # Stripe mock
      - "api.sendgrid.com:127.0.0.1"     # Email mock
      - "s3.amazonaws.com:192.168.1.50"  # MinIO local S3

      # Mock infrastructure services
      - "vault.corp.com:172.20.0.5"
      - "consul.corp.com:172.20.0.6"

    environment:
      # Application uses these URLs
      - STRIPE_API_URL=https://api.stripe.com
      - SENDGRID_URL=https://api.sendgrid.com
      # These now resolve to mock servers via /etc/hosts
```

## Step 5: extra_hosts in Docker CLI

For containers not managed by compose:

```bash
# Add host entry when running container
docker run -d \
  --add-host="database.internal:192.168.1.100" \
  --add-host="cache.internal:192.168.1.101" \
  --add-host="host.docker.internal:host-gateway" \
  --name myapp \
  myapp:latest

# Verify
docker exec myapp grep -E "database|cache|host.docker" /etc/hosts
```

## Step 6: Dynamic Hosts with Entrypoint Script

For environments where IPs change but a DNS server isn't available:

```dockerfile
# Dockerfile - Dynamic hosts from environment
FROM myapp:latest

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/sh
# entrypoint.sh - Add dynamic hosts at startup

# Add hosts from environment variables
if [ -n "$DB_IP" ]; then
    echo "$DB_IP  database.internal" >> /etc/hosts
fi

if [ -n "$CACHE_IP" ]; then
    echo "$CACHE_IP  cache.internal" >> /etc/hosts
fi

echo "=== /etc/hosts ==="
cat /etc/hosts

# Start the actual application
exec "$@"
```

```yaml
# docker-compose.yml using dynamic hosts
services:
  app:
    image: myapp-dynamic:latest
    environment:
      - DB_IP=10.0.1.10
      - CACHE_IP=10.0.1.11
    command: ["node", "server.js"]
```

## Conclusion

The `extra_hosts` directive is a lightweight alternative to running a DNS server when you need a small number of custom hostname mappings. It works reliably across restarts - entries are injected when the container starts. For development environments, override production API endpoints with local mocks. For production, map legacy hostnames to current infrastructure. Portainer displays container details including effective network configuration, helping you verify host entries without container exec access.
