# How to Automate SSL Certificate Renewal for Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, SSL, Let's Encrypt, Certbot, Traefik, Automation, Security

Description: Learn how to set up automatic SSL certificate renewal for Portainer and its managed services using Traefik or Certbot with automated renewal hooks.

---

SSL certificates expire every 90 days if you use Let's Encrypt. Manually renewing them is error-prone and easy to forget. This guide covers two approaches to automating SSL renewal for Portainer: using Traefik (which handles renewal automatically) and using Certbot with renewal hooks to reload Portainer.

---

## Approach 1: Traefik with Automatic SSL (Recommended)

Traefik handles certificate acquisition and renewal entirely automatically. Once configured, you never need to think about SSL again.

```yaml
# traefik-auto-ssl-stack.yml — Traefik with automatic Let's Encrypt
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
      # Redirect all HTTP to HTTPS
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      # ACME / Let's Encrypt configuration
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_letsencrypt:/letsencrypt

volumes:
  traefik_letsencrypt:
```

Traefik renews certificates automatically when they're within 30 days of expiry — no cron jobs needed.

---

## Approach 2: Certbot with Renewal Hooks

If you're using Nginx instead of Traefik, Certbot's renewal hooks can reload services after certificate renewal.

```bash
# Install Certbot
sudo apt update && sudo apt install -y certbot python3-certbot-nginx

# Obtain a certificate for Portainer's domain
sudo certbot certonly \
  --nginx \
  --email admin@example.com \
  --agree-tos \
  --no-eff-email \
  -d portainer.example.com

# Verify the certificate
sudo certbot certificates
```

---

### Create a Renewal Hook to Restart Portainer's Nginx

```bash
# /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
#!/bin/bash
# This script runs after every successful certificate renewal

echo "Certificate renewed. Reloading Nginx..."
systemctl reload nginx

# If Portainer is behind a Docker-managed Nginx container:
docker exec nginx nginx -s reload

echo "Reload complete at $(date)"
```

```bash
chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

---

### Test Certbot Auto-Renewal

```bash
# Dry-run renewal to verify the process works
sudo certbot renew --dry-run

# Check the systemd timer for automatic renewal
systemctl status certbot.timer

# View renewal logs
sudo journalctl -u certbot
```

---

## Approach 3: Portainer with Custom SSL Certificates

If you manage certificates externally, update Portainer to use them.

```bash
# Stop Portainer
docker stop portainer

# Copy new certificates to the Portainer data volume
docker run --rm \
  -v portainer_data:/data \
  -v /etc/letsencrypt:/certs \
  alpine \
  cp /certs/live/portainer.example.com/fullchain.pem /data/certs/cert.pem

docker run --rm \
  -v portainer_data:/data \
  -v /etc/letsencrypt:/certs \
  alpine \
  cp /certs/live/portainer.example.com/privkey.pem /data/certs/key.pem

# Start Portainer with custom cert paths
docker start portainer
```

---

## Monitoring Certificate Expiry with OneUptime

Set up an SSL expiry check in OneUptime to alert you if any certificate is within 14 days of expiry — as a safety net for your automation.

---

## Summary

The simplest path to automated SSL for Portainer is deploying Traefik as a reverse proxy. Traefik handles Let's Encrypt certificates and renewals automatically with zero maintenance. For Nginx-based setups, Certbot with renewal hooks in `/etc/letsencrypt/renewal-hooks/deploy/` provides reliable automation. Always test renewals with `--dry-run` before relying on them in production.
