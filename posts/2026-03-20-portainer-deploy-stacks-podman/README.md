# How to Deploy Stacks to Podman via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Stacks, Docker Compose, Deployment, Container Orchestration

Description: Learn how to deploy Docker Compose stacks to a Podman backend via Portainer, including compatibility notes and workarounds for Podman-specific differences.

---

Portainer can deploy Compose stacks to Podman environments just as it does for Docker. Podman 3.x+ includes a `podman-compose`-compatible API that handles most Docker Compose files without modification.

## Prerequisites

- Portainer connected to a Podman socket (see the connection guide)
- Podman 4.x or later (for best Compose API compatibility)
- `podman-compose` or the native Podman Compose support

## Deploying a Stack via Portainer

1. In Portainer go to **Stacks > Add Stack**.
2. Select the Podman environment.
3. Paste your Compose YAML or import from Git.
4. Click **Deploy the stack**.

Portainer translates the Compose spec into API calls to Podman.

## Example Stack Compatible with Podman

Most Docker Compose stacks work with Podman without changes:

```yaml
version: "3.8"

services:
  webapp:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - webapp_data:/usr/share/nginx/html

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: pgpass
      POSTGRES_DB: myapp
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  webapp_data:
  db_data:
```

## Known Compatibility Issues

**Network mode `host`:** Works on Podman rootful, but may fail on rootless Podman due to network namespace restrictions.

```yaml
# Workaround for rootless: use port mapping instead of host networking
services:
  app:
    ports:
      - "8080:80"     # Use explicit port mapping
    # network_mode: host  # Avoid this for rootless Podman
```

**Privileged containers:** Rootless Podman restricts privileged mode. You may need to add capabilities explicitly:

```yaml
services:
  app:
    cap_add:
      - NET_ADMIN
    security_opt:
      - no-new-privileges:false
```

**Volume ownership:** Rootless Podman uses user namespace mapping. Files inside volumes may show different UIDs. Set `PUID`/`PGID` environment variables if the image supports them.

## Using Podman Pods in Stacks

Podman can group Compose services into a pod for shared networking:

```yaml
# Portainer doesn't expose Podman pods natively
# But you can use podman-compose directly for pod-aware deployments:
podman-compose --pod-args "--hostname=mypod" up -d
```

## Monitoring Stack Health

Use OneUptime to monitor service endpoints in your Podman stacks just as you would with Docker stacks. The monitoring endpoint doesn't care whether Podman or Docker is running the container.
