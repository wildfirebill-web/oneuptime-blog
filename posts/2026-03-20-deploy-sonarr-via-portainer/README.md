# How to Deploy Sonarr via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Sonarr, Media Management, Docker, Self-Hosting, Automation

Description: Learn how to deploy Sonarr, the automated TV show download manager, via Portainer with proper volume mapping for media and download directories.

---

Sonarr monitors RSS feeds and automatically downloads new TV episodes via your download client (qBittorrent, Transmission, etc.). It renames and organizes files into your media library automatically. Portainer simplifies management of the Sonarr container.

## Prerequisites

- Portainer running
- A download client already deployed (qBittorrent or Transmission recommended)
- A media directory for finished TV shows

## Compose Stack

The key to a working Sonarr setup is mapping both the download directory and the final media directory so Sonarr can perform hardlink moves without copying across filesystems:

```yaml
version: "3.8"

services:
  sonarr:
    image: linuxserver/sonarr:latest
    restart: unless-stopped
    ports:
      - "8989:8989"
    environment:
      PUID: 1000    # Match to host user that owns media files
      PGID: 1000
      TZ: America/New_York
    volumes:
      - sonarr_config:/config
      # Map downloads and media on the same parent path so hardlinks work
      - /mnt/data/downloads:/downloads
      - /mnt/data/media/tv:/tv

volumes:
  sonarr_config:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `sonarr`.
3. Update `PUID`/`PGID` and volume paths to match your setup.
4. Click **Deploy the stack**.

Open `http://<host>:8989` to access the Sonarr UI.

## Connecting a Download Client

In Sonarr go to **Settings > Download Clients > Add**:

- Select qBittorrent or Transmission
- Set **Host** to the container name (e.g., `qbittorrent`) if on the same Docker network
- Set **Port** to the download client's API port
- Enter credentials and test the connection

## Adding Your First Series

1. Go to **Series > Add New**.
2. Search for a show name.
3. Set the root folder to `/tv`.
4. Choose a quality profile and click **Add Series**.

Sonarr will immediately start searching for missing episodes.

## Monitoring

Use OneUptime to monitor `http://<host>:8989/api/v3/health` (include `X-Api-Key` header from Sonarr's settings). This endpoint lists any active health warnings. A non-200 response means Sonarr itself is down.
