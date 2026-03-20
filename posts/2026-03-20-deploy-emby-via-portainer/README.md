# How to Deploy Emby via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Emby, Media Server, Docker, Self-Hosting, Streaming

Description: Learn how to deploy Emby media server via Portainer with media library mounts, hardware transcoding device passthrough, and persistent configuration.

---

Emby is a polished media server with a clean mobile interface and strong client ecosystem. The free tier supports most use cases; Emby Premiere unlocks hardware transcoding and additional features. Portainer makes the lifecycle management easy.

## Prerequisites

- Portainer running
- Media files on the host system
- At least 1GB RAM

## Compose Stack

```yaml
version: "3.8"

services:
  emby:
    image: emby/embyserver:latest
    restart: unless-stopped
    ports:
      - "8097:8096"     # HTTP port (offset to avoid conflict with Jellyfin)
      - "8921:8920"     # HTTPS port
    environment:
      UID: 1000         # Run as non-root user matching media file ownership
      GID: 1000
      GIDLIST: 44       # Add video group for GPU access (group ID may vary)
    volumes:
      - emby_config:/config
      - emby_cache:/cache
      - /mnt/media/movies:/mnt/movies:ro
      - /mnt/media/tv:/mnt/tv:ro
    # Uncomment for GPU hardware transcoding:
    # devices:
    #   - /dev/dri:/dev/dri

volumes:
  emby_config:
  emby_cache:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `emby`.
3. Adjust media paths and UID/GID to match your host user.
4. Click **Deploy the stack**.

Open `http://<host>:8097` to run the setup wizard. Add media libraries pointing to `/mnt/movies` and `/mnt/tv`.

## Hardware Transcoding

To enable hardware-accelerated transcoding (requires Emby Premiere):

1. Uncomment the `devices` block in the stack.
2. In Emby go to **Server Settings > Transcoding**.
3. Enable **Intel QuickSync** or **VAAPI** hardware acceleration.

## Remote Access

Emby provides built-in remote access via emby.media relay or port forwarding. For self-managed remote access, expose port `8097` through your firewall and optionally put it behind a reverse proxy with HTTPS.

## Monitoring

Use OneUptime to monitor `http://<host>:8097/System/Info/Public`. Emby returns a JSON object with server information when healthy. Alert on any non-200 responses to detect container crashes or database issues promptly.
