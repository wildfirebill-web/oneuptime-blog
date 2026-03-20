# How to Set Up Let's Encrypt ACME with Traefik for Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Traefik, Let's Encrypt, ACME, TLS

Description: Step-by-step guide to configuring Traefik's ACME integration with Let's Encrypt to automatically issue and renew free TLS certificates for Portainer and other services.

## Prerequisites

- Docker and Docker Compose installed
- A domain name with DNS pointing to your server
- Port 80 accessible from the internet (for HTTP challenge)

## Step 1: Prepare Certificate Storage

```bash
# Create directories
mkdir -p /opt/traefik/data

# Create and lock the ACME storage file
touch /opt/traefik/data/acme.json
chmod 600 /opt/traefik/data/acme.json
```

This file stores all issued certificates and must be readable only by Traefik.

## Step 2: Traefik Static Configuration

```yaml
# /opt/traefik/traefik.yml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      # Your email for expiration warnings
      email: admin@yourdomain.com
      # Where to store certificates
      storage: /data/acme.json
      # HTTP-01 challenge (requires port 80)
      httpChallenge:
        entryPoint: web

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: proxy

api:
  dashboard: true

log:
  level: INFO
```

## Step 3: Docker Compose with Portainer

```yaml
# docker-compose.yml
version: "3.8"

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    networks:
      - proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/traefik/traefik.yml:/traefik.yml:ro
      - /opt/traefik/data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik-dashboard.rule=Host(`traefik.yourdomain.com`)"
      - "traefik.http.routers.traefik-dashboard.entrypoints=websecure"
      - "traefik.http.routers.traefik-dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.traefik-dashboard.service=api@internal"

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    networks:
      - proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - portainer_data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.yourdomain.com`)"
      - "traefik.http.routers.portainer.entrypoints=websecure"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"

networks:
  proxy:
    name: proxy

volumes:
  portainer_data:
```

## Step 4: Test with Let's Encrypt Staging First

Before using the production Let's Encrypt server, test with the staging server to avoid rate limit issues:

```yaml
certificatesResolvers:
  letsencrypt-staging:
    acme:
      email: admin@yourdomain.com
      storage: /data/acme-staging.json
      caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
      httpChallenge:
        entryPoint: web
```

Then replace `letsencrypt` with `letsencrypt-staging` in your container labels while testing.

## Step 5: Deploy and Monitor

```bash
cd /opt/traefik
docker compose up -d

# Watch for certificate issuance (takes ~30 seconds)
docker logs traefik --follow 2>&1 | grep -E "acme|cert|error"
```

Successful issuance looks like:

```
level=debug msg="Obtaining certificate for domains \"portainer.yourdomain.com\""
level=info msg="Certificate obtained successfully"
```

## Verifying the Certificate

```bash
# Check the certificate details
echo | openssl s_client -servername portainer.yourdomain.com \
  -connect portainer.yourdomain.com:443 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates

# Output should show:
# issuer=C=US, O=Let's Encrypt, CN=R11
```

## Let's Encrypt Rate Limits

| Limit | Value |
|-------|-------|
| Certificates per registered domain | 50 per week |
| Duplicate certificates | 5 per week |
| Failed validations | 5 per account per hostname per hour |

Always test with staging to avoid hitting production rate limits.

## Conclusion

Traefik's Let's Encrypt integration makes HTTPS effortless — once configured, any container running behind Traefik via Portainer gets a valid certificate automatically, with no manual renewal process or CA cost.
