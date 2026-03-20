# How to Disable HTTP Access in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, HTTP, HTTPS, Docker

Description: Learn how to completely disable plain HTTP access to the Portainer UI and enforce HTTPS-only connections.

---

Allowing HTTP access to Portainer in production exposes your credentials and session tokens to network interception. Disabling HTTP is a critical security hardening step.

## Understanding Portainer's HTTP Ports

Portainer can expose two web interfaces:
- **Port 9443**: HTTPS (TLS encrypted) - recommended
- **Port 9000**: HTTP (unencrypted) - should be disabled in production

The key insight: port 9000 is only accessible if you explicitly map it with `-p 9000:9000` in your Docker run command. Simply omitting this mapping effectively disables HTTP.

## Step 1: Remove HTTP Port Mapping

Stop and recreate Portainer without the HTTP port:

```bash
# Stop and remove current container

docker stop portainer
docker container rm portainer

# Restart WITHOUT -p 9000:9000
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 2: Block Port 9000 at the Firewall Level

Even if Portainer isn't publishing port 9000, add a firewall rule as defense in depth:

```bash
# Block port 9000 with ufw
sudo ufw deny 9000/tcp
sudo ufw deny 9000/udp

# Verify the rule
sudo ufw status | grep 9000
```

## Step 3: Update Portainer Agent Configurations

If you're running Portainer Agent behind the server, ensure agents aren't using HTTP tunnels:

```bash
# Verify agents use TLS (port 9001 by default)
# Portainer Agent runs on port 9001, NOT 9000
docker ps --filter name=portainer_agent --format "table {{.Names}}\t{{.Ports}}"
```

## Step 4: Configure Nginx to Block HTTP

If you're using a reverse proxy, add an explicit HTTP deny rule:

```nginx
# Block HTTP access to Portainer backend
server {
    listen 80;
    server_name portainer.example.com;

    # Return 403 rather than redirecting (stricter)
    return 403 "HTTP access disabled. Use HTTPS.";
}
```

Or to redirect to HTTPS instead:

```nginx
server {
    listen 80;
    server_name portainer.example.com;
    return 301 https://$host$request_uri;
}
```

## Verify HTTP is Disabled

```bash
# Test that HTTP port 9000 is not reachable
nc -zv localhost 9000 2>&1
# Expected: Connection refused

# Test that HTTPS port 9443 still works
curl -k -o /dev/null -w "%{http_code}" https://localhost:9443/
# Expected: 200
```

## Docker Compose Example

```yaml
# docker-compose.yml - HTTP disabled
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "8000:8000"   # Edge agent tunnel
      - "9443:9443"   # HTTPS only - port 9000 not exposed
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

---

*Monitor Portainer's HTTPS endpoint with [OneUptime](https://oneuptime.com) SSL certificate monitoring.*
