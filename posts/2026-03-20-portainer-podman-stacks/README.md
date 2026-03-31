# How to Deploy Stacks to Podman via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Docker Compose, Stack, Self-Hosted

Description: Deploy Docker Compose stacks to Podman environments using Portainer, leveraging Podman's Docker-compatible API for stack management.

## Introduction

With Podman's Docker-compatible API and `podman-compose` support, Portainer can deploy and manage multi-container stacks on Podman-backed environments. This guide shows how to deploy stacks to Podman via Portainer with the appropriate configuration for Podman's networking and storage model.

## Prerequisites

- Portainer connected to a Podman socket
- `podman-compose` installed on the Podman host
- Podman 4.0+

## Step 1: Install podman-compose

```bash
# Install podman-compose on the Podman host

pip3 install podman-compose

# Or via package manager
sudo dnf install podman-compose    # RHEL/Fedora
sudo apt install podman-compose    # Ubuntu/Debian (may need PPA)

# Verify
podman-compose --version
```

## Step 2: Create a Stack in Portainer

Navigate to **Stacks** → **Add Stack** → **Web Editor** in Portainer (when connected to a Podman environment):

```yaml
version: "3.8"
services:
  # Web application
  webapp:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - webapp_data:/usr/share/nginx/html
    networks:
      - app-net

  # Database
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppassword
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app-net

volumes:
  webapp_data:
  db_data:

networks:
  app-net:
    driver: bridge
```

## Step 3: Podman-Specific Compose Considerations

Podman Compose has some differences from Docker Compose:

```yaml
# For rootless Podman, user namespace mapping may be needed
services:
  myapp:
    image: myapp:latest
    # Podman-specific: run as specific user
    user: "1000:1000"
    # Podman uses --userns=keep-id for rootless
    security_opt:
      - label:disable   # Disable SELinux labels (or configure properly)

# Podman doesn't support --network=host the same way in rootless mode
# Use slirp4netns for rootless networking:
networks:
  default:
    driver: bridge
```

## Step 4: Deploy and Monitor

1. Enter the stack name and compose file in Portainer
2. Click **Deploy the stack**
3. Portainer will use the connected Podman environment to deploy the stack

```bash
# Verify deployment on Podman host
podman ps
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

## Step 5: Using Podman Pods Instead of Individual Containers

Podman's native concept is pods (like Kubernetes pods):

```yaml
# While Portainer uses Docker Compose format,
# the underlying Podman can use pod-based deployment

# Create a pod on Podman host
podman pod create --name myapp-pod -p 8080:80 -p 5432:5432

# Then Portainer manages containers that run in this pod
```

## Step 6: Volume Management in Podman

```yaml
# Podman volumes work similarly to Docker volumes
volumes:
  mydata:
    # Optional: specify Podman driver
    driver: local
    driver_opts:
      type: none
      device: /opt/myapp/data  # Bind to host path
      o: bind

# Or use Podman's container storage
volumes:
  mydata:  # Uses Podman's default storage
```

```bash
# List Podman volumes
podman volume ls

# Inspect a volume
podman volume inspect mydata

# Create volume manually
podman volume create mydata
```

## Step 7: Networking in Rootless Podman

```yaml
# Rootless Podman uses CNI or netavark for networking
# Port mapping works differently for rootless:

services:
  webapp:
    image: nginx:alpine
    ports:
      # Rootless Podman can only bind to ports > 1024
      - "8080:80"    # OK for rootless
      # - "80:80"    # NOT allowed without special config

# To allow ports < 1024 in rootless:
# echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee -a /etc/sysctl.conf
# sudo sysctl -p
```

## Step 8: Persistent Stacks Configuration

```bash
# Export stack files that work with both Docker and Podman
# The Docker Compose v3 format is compatible

# Deploy directly with podman-compose
cd /opt/stacks/myapp
podman-compose up -d

# List running stacks
podman-compose ps

# View logs
podman-compose logs -f
```

## Step 9: Healthchecks in Podman

```yaml
# Healthchecks work the same as Docker
services:
  webapp:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

```bash
# Check health status in Podman
podman inspect webapp-container | jq '.[0].State.Health.Status'
```

## Step 10: Handle Podman SELinux Labels

On SELinux-enabled systems:

```yaml
# For volumes mounted from host paths, add SELinux labels
services:
  myapp:
    volumes:
      - /host/path:/container/path:z    # Shared label
      # or
      - /host/path:/container/path:Z    # Private label
```

## Conclusion

Deploying stacks to Podman via Portainer uses the same Docker Compose format, with some considerations for rootless Podman networking (ports above 1024) and SELinux volume labels. Portainer communicates with Podman through its Docker-compatible API, so most stack operations work transparently. For production Podman deployments, pay special attention to rootless vs rootful mode and the networking implications for port binding.
