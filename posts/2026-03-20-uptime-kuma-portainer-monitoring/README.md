# How to Set Up Uptime Kuma via Portainer for Service Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Uptime Kuma, Monitoring, Self-Hosted, Uptime

Description: Deploy Uptime Kuma through Portainer to monitor the uptime and availability of your websites, APIs, and services with a beautiful dashboard.

## Introduction

Uptime Kuma is a self-hosted monitoring tool that tracks the availability of websites, APIs, TCP ports, DNS records, and more. Deploying it via Portainer takes only a few minutes and gives you a polished status page without any cloud subscription.

## Prerequisites

- Docker host with Portainer installed
- Port 3001 available on the host
- Basic Portainer knowledge

## Deploying Uptime Kuma

### Step 1: Create a Volume

In Portainer, navigate to **Volumes** and click **Add volume**:

- Name: `uptime-kuma-data`
- Driver: `local`

Click **Create the volume**.

### Step 2: Deploy as a Container

Navigate to **Containers > Add container**:

- **Name**: `uptime-kuma`
- **Image**: `louislam/uptime-kuma:1`
- **Port mapping**: Host `3001` → Container `3001`
- **Volumes**: Map `uptime-kuma-data` to `/app/data`
- **Restart policy**: `Unless stopped`

Click **Deploy the container**.

### Step 3: Deploy Using a Stack (Recommended)

For a more reproducible setup, use a Portainer stack. Navigate to **Stacks > Add stack** and paste:

```yaml
version: "3.8"

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    # Persist monitoring data between restarts
    volumes:
      - uptime-kuma-data:/app/data
      # Optionally mount Docker socket to monitor Docker containers
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "3001:3001"
    restart: unless-stopped
    # Security: run with limited privileges
    security_opt:
      - no-new-privileges:true

volumes:
  uptime-kuma-data:
    driver: local
```

Click **Deploy the stack**.

### Step 4: Initial Setup

1. Open `http://<your-host>:3001` in a browser
2. Create your admin account on first launch
3. You'll land on the main dashboard

### Step 5: Add Your First Monitor

Click **Add New Monitor** and configure:

| Field | Example Value |
|-------|---------------|
| Monitor Type | HTTP(s) |
| Friendly Name | My Website |
| URL | https://example.com |
| Heartbeat Interval | 60 seconds |
| Retries | 3 |

Click **Save** and Uptime Kuma will immediately start checking.

### Step 6: Configure Notifications

Navigate to **Settings > Notifications** to set up alerts. Uptime Kuma supports:

- **Slack** - webhook URL
- **Discord** - webhook URL
- **Telegram** - bot token + chat ID
- **Email (SMTP)** - configure SMTP credentials
- **PagerDuty** - integration key
- **Pushover**, **Gotify**, and 80+ more

### Step 7: Set Up a Status Page

1. Click **Status Page** in the sidebar
2. Click **New Status Page**
3. Give it a name and slug (e.g., `status`)
4. Add monitors to the page
5. Access at `http://<your-host>:3001/status/<slug>`

## Advanced: Enable HTTPS with a Reverse Proxy

If Portainer is behind Traefik, add these labels to the stack:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    volumes:
      - uptime-kuma-data:/app/data
    labels:
      # Traefik labels for automatic HTTPS
      - "traefik.enable=true"
      - "traefik.http.routers.uptime-kuma.rule=Host(`status.example.com`)"
      - "traefik.http.routers.uptime-kuma.entrypoints=websecure"
      - "traefik.http.routers.uptime-kuma.tls.certresolver=letsencrypt"
      - "traefik.http.services.uptime-kuma.loadbalancer.server.port=3001"
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

## Updating Uptime Kuma

In Portainer, navigate to your stack, click **Editor**, update the image tag, and click **Update the stack**. Portainer will pull the new image and recreate the container while preserving your data volume.

## Conclusion

Uptime Kuma deployed via Portainer gives you a professional monitoring solution in minutes. With support for dozens of monitor types and notification channels, it's an excellent choice for home labs and small production environments. The persistent volume ensures your monitoring history survives container updates.
