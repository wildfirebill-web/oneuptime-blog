# How to Deploy Vaultwarden (Bitwarden) via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Vaultwarden, Bitwarden, Password Manager, Self-Hosted, Security

Description: Deploy Vaultwarden (a Bitwarden-compatible server) via Portainer for a self-hosted password manager that works with all official Bitwarden clients.

## Introduction

Vaultwarden is an unofficial, lightweight implementation of the Bitwarden server API written in Rust. It's compatible with all official Bitwarden clients (browser extensions, desktop apps, mobile apps) while using significantly less resources than the official server. Deploy via Portainer for a private, self-hosted password manager.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    environment:
      # Required: Set your domain URL
      DOMAIN: https://vault.example.com
      
      # Admin panel (generate token with: openssl rand -hex 32)
      ADMIN_TOKEN: your_generated_admin_token
      
      # Disable user registration after initial setup
      SIGNUPS_ALLOWED: "true"   # Set to false after creating your account
      
      # Email settings
      SMTP_HOST: smtp.example.com
      SMTP_FROM: vaultwarden@example.com
      SMTP_PORT: 587
      SMTP_SECURITY: starttls
      SMTP_USERNAME: vaultwarden@example.com
      SMTP_PASSWORD: smtp_password
      
      # Security
      INVITATION_ORG_NAME: "Your Organization"
      
      # Performance
      ROCKET_WORKERS: 2
    volumes:
      - vaultwarden_data:/data
    ports:
      - "8080:80"    # HTTP (put behind HTTPS reverse proxy)
    restart: unless-stopped

volumes:
  vaultwarden_data:
```

## HTTPS Requirement

Vaultwarden **requires HTTPS** for the browser extensions to work. Use Traefik or Caddy as a reverse proxy:

### With Traefik

```yaml
services:
  vaultwarden:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vaultwarden.rule=Host(`vault.example.com`)"
      - "traefik.http.routers.vaultwarden.entrypoints=websecure"
      - "traefik.http.routers.vaultwarden.tls.certresolver=letsencrypt"
      - "traefik.http.services.vaultwarden.loadbalancer.server.port=80"
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

### With Caddy

```caddyfile
vault.example.com {
    reverse_proxy vaultwarden:80
}
```

## Initial Setup

1. Access `https://vault.example.com` and create your account
2. Access the admin panel at `https://vault.example.com/admin` using your ADMIN_TOKEN
3. After creating your account(s), set `SIGNUPS_ALLOWED: "false"` in the stack

## Configuring Bitwarden Clients

### Browser Extension

1. Click the **Bitwarden** extension
2. Click **Settings** (gear icon)
3. Set **Server URL**: `https://vault.example.com`
4. Log in with your account

### Desktop App (Windows/Mac/Linux)

1. Open Bitwarden
2. Click the region selector at the top-left
3. Set **Self-hosted** and enter `https://vault.example.com`
4. Log in

### Mobile App (iOS/Android)

1. Open Bitwarden
2. Tap the region selector on the login screen
3. Select **Self-hosted**
4. Enter `https://vault.example.com`
5. Log in

## Enabling 2FA

1. In Vaultwarden, go to **Account Settings > Security > Two-step Login**
2. Enable TOTP authenticator
3. Scan QR code with your authenticator app

## Backup Vaultwarden

```bash
# Backup Vaultwarden data

docker stop vaultwarden

docker run --rm \
  -v vaultwarden_data:/source \
  -v /backups:/backup \
  alpine tar czf /backup/vaultwarden-$(date +%Y%m%d).tar.gz /source

docker start vaultwarden
```

## Conclusion

Vaultwarden deployed via Portainer gives you a private, self-hosted password manager that's fully compatible with all official Bitwarden clients. HTTPS is required for browser extension functionality, making a Traefik or Caddy reverse proxy essential. Once set up, it provides the same experience as Bitwarden cloud at zero subscription cost, with complete control over your password data.
