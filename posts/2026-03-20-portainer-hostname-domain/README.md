# How to Set Container Hostname and Domain in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Networking, DevOps

Description: Learn how to configure custom hostnames and domain names for Docker containers in Portainer for service identification and internal DNS resolution.

## Introduction

Docker containers are assigned random hostnames by default (usually the short container ID). For services that rely on their hostname for configuration, logging, or SSL certificates, setting a meaningful hostname is important. Portainer lets you configure both the hostname and domain name for containers through the web UI.

## Prerequisites

- Portainer installed with a connected Docker environment

## How Container Hostname Works

Inside a container, `hostname` returns the container's hostname. This value:
- Appears in log messages and application output
- Is used by some applications for cluster membership
- May be referenced in SSL certificate generation
- Is separate from the container's Docker network name (which is the container name)

## Step 1: Set Hostname in Portainer

1. Navigate to **Containers > Add container**.
2. Set the container name and image.
3. Scroll to the **Network** tab.
4. Find the **Hostname** field.
5. Enter your desired hostname.

```text
# Examples:

Hostname: web-server-01
Hostname: api-gateway
Hostname: db-primary
Hostname: app-prod-berlin-01
```

## Step 2: Set Domain Name

The domain name is appended to the hostname to form the FQDN (Fully Qualified Domain Name):

```text
Hostname: app-server
Domain:   internal.example.com

# Results in FQDN: app-server.internal.example.com
# /etc/hosts inside container: 172.17.0.2 app-server.internal.example.com app-server
```

In Portainer, find the **Domain name** field in the Network tab.

## Step 3: Add Custom Host Entries

Portainer also lets you add entries to the container's `/etc/hosts` file:

```text
# Custom host entries (extra_hosts in compose):
IP:           10.0.0.50
Hostname:     database.internal
```

This is useful when you can't rely on DNS and need specific IP-to-hostname mappings.

In docker-compose.yml:

```yaml
services:
  app:
    image: myorg/app:latest
    extra_hosts:
      - "database.internal:10.0.0.50"
      - "cache.internal:10.0.0.51"
      - "api.external.com:203.0.113.100"
```

## Step 4: Full Configuration Example

```yaml
# docker-compose.yml with hostname, domain, and custom hosts
version: "3.8"

services:
  web:
    image: nginx:alpine
    hostname: web-server-01
    domainname: internal.mycompany.com
    # FQDN will be: web-server-01.internal.mycompany.com
    extra_hosts:
      - "db-primary:10.0.1.10"
      - "db-replica:10.0.1.11"
      - "redis-cluster:10.0.1.20"
    restart: unless-stopped

  app:
    image: myorg/myapp:latest
    hostname: app-${INSTANCE_ID:-001}
    domainname: prod.mycompany.com
    extra_hosts:
      - "monitoring.internal:10.0.0.100"
```

## Step 5: Common Use Cases

### Applications That Use Hostname in Configuration

Some applications (like RabbitMQ, Cassandra, Kafka) use the hostname for cluster membership:

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    # RabbitMQ uses hostname for cluster node name
    hostname: rabbitmq-node-1
    domainname: mycluster.internal
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASSWORD}
      # Use the hostname for the cluster node name
      - RABBITMQ_NODENAME=rabbit@rabbitmq-node-1
```

### SSL/TLS Certificates

When generating certificates inside a container based on hostname:

```yaml
services:
  cert-service:
    image: myorg/cert-generator:latest
    hostname: api-gateway
    domainname: api.mycompany.com
    # Container generates SSL cert for: api-gateway.api.mycompany.com
    command: generate-cert --cn api-gateway.api.mycompany.com
```

### Log Aggregation Identification

Set a meaningful hostname so log entries are identifiable:

```yaml
services:
  app:
    image: myorg/app:latest
    # Logs will show this hostname instead of a random container ID
    hostname: prod-app-eu-west-01
    environment:
      - LOG_FORMAT=json
      # App uses hostname in log output
```

### Database Replication

```yaml
services:
  postgres-primary:
    image: postgres:15
    hostname: pg-primary
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      # PostgreSQL replication uses hostname for identification
      - POSTGRES_HOST_AUTH_METHOD=md5

  postgres-replica:
    image: postgres:15
    hostname: pg-replica
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_PRIMARY_HOST=pg-primary  # Connect to primary by hostname
```

## Step 6: Verify Hostname Configuration

Inside the running container:

```bash
# In the container console (via Portainer Exec):

# Check hostname:
hostname
# Output: web-server-01

# Check FQDN:
hostname -f
# Output: web-server-01.internal.mycompany.com

# Check /etc/hosts:
cat /etc/hosts
# 127.0.0.1 localhost
# ::1 localhost
# 172.17.0.2 web-server-01.internal.mycompany.com web-server-01
# 10.0.0.50 database.internal  (from extra_hosts)
```

## Hostname vs. Container Name

Important distinction:

| Setting | Purpose | Visible to |
|---------|---------|------------|
| **Container name** | Docker daemon identity; service discovery in Docker networks | Docker (service DNS: `container-name`) |
| **Hostname** | Inside the container's OS | Application code, logs, hostname command |
| **Domain name** | Appended to hostname | Inside the container's /etc/hosts |

```bash
# Container name (service discovery):
ping my-web-container   # Works from other containers on the same network

# Hostname (inside the container):
docker exec my-web-container hostname
# Output: web-server-01 (the configured hostname, not the container name)
```

## Conclusion

Setting a custom hostname and domain name in Portainer ensures containers are properly identified in logs, applications that rely on hostname-based configuration work correctly, and SSL certificates can be generated for meaningful domain names. Combined with custom `/etc/hosts` entries, you have full control over how containers identify themselves and how they resolve internal hostnames without requiring changes to DNS infrastructure.
