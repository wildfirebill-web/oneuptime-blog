# How to Set Up Portainer Behind Caddy Reverse Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Caddy, Reverse Proxy, HTTPS

Description: Use Caddy as a reverse proxy for Portainer with automatic HTTPS certificate management and a minimal configuration file.

## Introduction

Caddy is renowned for its simplicity - it automatically obtains and renews TLS certificates from Let's Encrypt with zero configuration. Setting up Portainer behind Caddy takes just a few lines in a Caddyfile, making it an excellent choice for teams who want HTTPS without complexity.

## Prerequisites

- Docker and Docker Compose installed
- A domain pointing to your server (Caddy uses it for ACME challenges)
- Ports 80 and 443 open

## Step 1: Create the Caddyfile

Create `caddy/Caddyfile`:

```caddyfile
# Caddy automatically handles HTTPS - just provide the domain

portainer.example.com {
    # Reverse proxy to Portainer's HTTPS port
    reverse_proxy portainer:9443 {
        # Trust Portainer's self-signed certificate
        transport http {
            tls_insecure_skip_verify
        }
    }

    # Set forwarded headers so Portainer knows the real client
    header_up X-Forwarded-Proto {scheme}
    header_up X-Forwarded-Host {host}
    header_up X-Real-IP {remote_host}

    # Security headers
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
    }
}
```

Alternatively, if Portainer is started with `--http-enabled`:

```caddyfile
portainer.example.com {
    # Proxy to HTTP port - simpler, no TLS verification needed
    reverse_proxy portainer:9000

    header_up X-Forwarded-Proto {scheme}
}
```

## Step 2: Create the Docker Compose File

```yaml
version: "3.8"

services:
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: always
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"   # HTTP/3 support
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data       # Stores certificates
      - caddy_config:/config   # Stores Caddy config
    networks:
      - proxy

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - proxy
    command:
      - "--trusted-origins=https://portainer.example.com"

networks:
  proxy:
    driver: bridge

volumes:
  portainer_data:
  caddy_data:
  caddy_config:
```

## Step 3: Deploy the Stack

```bash
# Start both services
docker compose up -d

# Watch Caddy obtain the certificate (takes 30-60 seconds)
docker logs caddy --follow

# Once you see "certificate obtained", verify HTTPS
curl -I https://portainer.example.com
```

## Step 4: Validate the Setup

```bash
# Check Caddy's certificate status
docker exec caddy caddy list-modules

# Reload Caddy config without restart
docker exec caddy caddy reload --config /etc/caddy/Caddyfile

# View Caddy access logs
docker logs caddy 2>&1 | grep portainer
```

## Using Caddy with Docker Labels (Automatic Discovery)

For a more dynamic setup, use the `caddy-docker-proxy` plugin:

```yaml
  portainer:
    image: portainer/portainer-ce:latest
    labels:
      caddy: portainer.example.com
      caddy.reverse_proxy: "{{upstreams 9443}}"
      caddy.reverse_proxy.transport: http
      caddy.reverse_proxy.transport.tls_insecure_skip_verify: ""
```

## Troubleshooting

**Certificate not obtained**: Ensure DNS is pointing to your server and ports 80/443 are accessible. Caddy uses HTTP-01 challenge on port 80 first.

**Trusted origins error**: Add `--trusted-origins=https://portainer.example.com` to Portainer's command.

**Caddy using staging certificates**: Check `/data/caddy/certificates` - if you see `acme-staging`, Caddy hit rate limits. Wait and force renewal.

## Conclusion

Caddy makes HTTPS with reverse proxying trivially simple. A three-line Caddyfile gives you automatic certificate management, HTTP/2, and proper proxy headers. For teams wanting the simplest possible setup with automatic TLS, Caddy is often the best choice.
