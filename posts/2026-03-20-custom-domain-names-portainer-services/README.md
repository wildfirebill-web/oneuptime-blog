# How to Configure Custom Domain Names for Portainer Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, DNS, Nginx, Traefik, Docker, Reverse Proxy, Domain

Description: Learn how to configure custom domain names for services managed by Portainer using Nginx or Traefik as a reverse proxy with automatic SSL.

---

Running services on `IP:port` combinations is functional but impractical. Custom domain names like `app.example.com` are easier to share, work with SSL certificates, and integrate cleanly with auth middleware. This guide shows how to configure custom domains for Portainer-managed containers using Nginx Proxy Manager (NPM) or Traefik.

---

## Approach 1: Nginx Proxy Manager (Recommended for Simplicity)

Nginx Proxy Manager provides a GUI for managing reverse proxy rules and automatic Let's Encrypt SSL.

### Deploy NPM as a Portainer Stack

```yaml
# nginx-proxy-manager-stack.yml

version: "3.8"

services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"      # HTTP
      - "443:443"    # HTTPS
      - "81:81"      # NPM Admin UI
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt

volumes:
  npm_data:
  npm_letsencrypt:
```

---

### Configure a Proxy Host in NPM

1. Open NPM at `http://<server-ip>:81`
2. Log in (default: `admin@example.com` / `changeme`)
3. Go to **Proxy Hosts > Add Proxy Host**
4. Fill in:
   - **Domain Names**: `app.example.com`
   - **Forward Hostname/IP**: container name or IP (e.g., `myapp` or `172.18.0.5`)
   - **Forward Port**: container port (e.g., `8080`)
5. On the **SSL** tab: enable **Request a new SSL Certificate** with Let's Encrypt
6. Click **Save**

---

## Approach 2: Traefik (Recommended for Automation)

Traefik reads Docker labels from containers and auto-creates routing rules.

### Deploy Traefik as a Portainer Stack

```yaml
# traefik-stack.yml
version: "3.8"

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_letsencrypt:/letsencrypt
    networks:
      - proxy

networks:
  proxy:
    external: true

volumes:
  traefik_letsencrypt:
```

```bash
# Create the shared proxy network before deploying
docker network create proxy
```

---

### Add Traefik Labels to Your App Container

```yaml
# myapp-stack.yml - with Traefik routing labels
version: "3.8"

services:
  myapp:
    image: myapp:latest
    restart: unless-stopped
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      # Route app.example.com to this container
      - "traefik.http.routers.myapp.rule=Host(`app.example.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
      - "traefik.http.services.myapp.loadbalancer.server.port=8080"

networks:
  proxy:
    external: true
```

---

## DNS Setup

Point your domain to your server's public IP. In Cloudflare or your DNS provider:

```text
app.example.com    A    203.0.113.10
```

For local-only domains, use Pi-hole or AdGuard Home DNS rewrites pointing to your LAN IP instead.

---

## Summary

Custom domain names for Portainer services require two components: a reverse proxy (NPM or Traefik) to route domains to containers, and DNS records pointing the domain to your server's IP. NPM is easier for manual setups; Traefik automates routing via container labels and is ideal for teams managing many services.
