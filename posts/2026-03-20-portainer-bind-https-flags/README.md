# How to Use the --bind and --bind-https Flags in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, CLI, Configuration, Networking, HTTPS

Description: Configure Portainer's HTTP and HTTPS listening addresses using the --bind and --bind-https flags to control which network interfaces and ports Portainer binds to.

## Introduction

By default, Portainer listens on all interfaces (`0.0.0.0`) on ports 9000 (HTTP) and 9443 (HTTPS). The `--bind` and `--bind-https` flags let you change these defaults — binding to a specific interface, a non-standard port, or restricting to localhost for security.

## Understanding the Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `--bind` | `0.0.0.0:9000` | HTTP listen address |
| `--bind-https` | `0.0.0.0:9443` | HTTPS listen address |

These flags accept an `address:port` format.

## Step 1: Bind to a Specific Interface

```bash
# Listen only on localhost (no external access)
docker run -d \
  -p 127.0.0.1:9000:9000 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --bind=0.0.0.0:9000   # Inside container binds all interfaces

# Or bind only to localhost inside the container
docker run -d \
  -p 9000:9000 \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --bind=0.0.0.0:9000    # Container internal address
```

**Note**: The `--bind` flag controls what the Portainer process listens on inside the container. To restrict external access, use Docker's port binding (`-p 127.0.0.1:9000:9000`).

## Step 2: Change the Default Port

```bash
# Run Portainer HTTP on port 8080 (inside container)
docker run -d \
  -p 8080:8080 \      # Map host port 8080 to container port 8080
  -p 8443:8443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --bind=0.0.0.0:8080 \
  --bind-https=0.0.0.0:8443
```

## Step 3: HTTPS-Only with Custom Port

```bash
# Run HTTPS only on port 443 inside container
docker run -d \
  -p 443:443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /opt/portainer/certs:/certs \
  portainer/portainer-ce:latest \
  --bind-https=0.0.0.0:443 \
  --http-disabled \
  --ssl \
  --sslcert /certs/cert.pem \
  --sslkey /certs/key.pem
```

## Step 4: Bind to a Specific IP Inside the Container

```bash
# Bind to a specific Docker network IP
# First, check the container's IP
docker run -it --rm alpine ip addr

# Typically on docker0 bridge: 172.17.0.x

# Bind Portainer to specific IP inside container (advanced)
docker run -d \
  -p 9000:9000 \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --bind=172.17.0.2:9000   # Specific container IP
```

## Step 5: Use Custom Ports with Docker Compose

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    # Map custom internal ports to host
    ports:
      - "8080:8080"   # HTTP on 8080
      - "8443:8443"   # HTTPS on 8443
    # Override the default bind addresses
    command: >
      --bind=0.0.0.0:8080
      --bind-https=0.0.0.0:8443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
```

## Step 6: Localhost-Only for Reverse Proxy Use

When using a reverse proxy (Nginx/Traefik), you may want Portainer to only listen on localhost:

```bash
# Run Portainer listening only on 127.0.0.1 inside container
# External access goes through the reverse proxy only
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  --network host \   # Use host networking for localhost binding to work
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --bind=127.0.0.1:9000 \
  --bind-https=127.0.0.1:9443
```

This ensures Portainer is only accessible via the reverse proxy.

## Step 7: Verify the Binding

```bash
# Check what Portainer is actually listening on
ss -tlnp | grep portainer

# Or check the container port bindings
docker inspect portainer | jq '.[0].HostConfig.PortBindings'

# Test connectivity to each configured address
curl -v http://localhost:9000/api/status
curl -kv https://localhost:9443/api/status
```

## Step 8: Combine with Other Startup Flags

```bash
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  -p 8000:8000 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --bind=0.0.0.0:9000 \               # HTTP
  --bind-https=0.0.0.0:9443 \         # HTTPS
  --snapshot-interval=300 \            # Snapshot every 5 minutes
  --admin-password="$HASHED_PASS" \   # Set admin password
  --tunnel-port=8000                   # Edge agent tunnel port
```

## Conclusion

The `--bind` and `--bind-https` flags give you precise control over which interfaces and ports Portainer listens on inside the container. The most common use case is changing the default port to avoid conflicts, or binding to all interfaces (`0.0.0.0`) in combination with Docker's `-p` port mapping to control external accessibility. For reverse proxy deployments, combine host networking with localhost binding to ensure Portainer is only reachable through the proxy.
