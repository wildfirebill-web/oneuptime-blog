# How to Deploy Prowlarr via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Prowlarr, Indexer Manager, Docker, Self-Hosting, Sonarr, Radarr

Description: Learn how to deploy Prowlarr, the unified indexer manager for Sonarr, Radarr, and Lidarr, via Portainer for centralized torrent and Usenet indexer management.

---

Prowlarr acts as a single place to manage all your indexers (torrent sites and Usenet providers) and syncs them automatically to Sonarr, Radarr, and Lidarr. Without Prowlarr you would have to add each indexer individually to every application.

## Prerequisites

- Portainer running
- Sonarr and/or Radarr already deployed on the same Docker network

## Compose Stack

```yaml
version: "3.8"

services:
  prowlarr:
    image: linuxserver/prowlarr:latest
    restart: unless-stopped
    ports:
      - "9696:9696"
    environment:
      PUID: 1000
      PGID: 1000
      TZ: America/New_York
    volumes:
      - prowlarr_config:/config

volumes:
  prowlarr_config:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `prowlarr`.
3. Click **Deploy the stack**.

Open `http://<host>:9696`.

## Adding Indexers

1. In Prowlarr go to **Indexers > Add Indexer**.
2. Search for your indexer (e.g., "1337x", "EZTV", "NZBGeek").
3. Enter your credentials and test the connection.
4. Repeat for each indexer.

## Syncing to Sonarr and Radarr

In Prowlarr go to **Settings > Apps**:

```
# For each *arr application:
App:            Sonarr (or Radarr)
Prowlarr Server: http://prowlarr:9696
App Server:     http://sonarr:8989
API Key:        <copy from Sonarr's Settings > General>
Sync Level:     Full Sync
```

After saving, all indexers in Prowlarr propagate to every connected application instantly.

## FlareSolverr Integration

Many indexers use Cloudflare challenges. Add FlareSolverr to bypass them:

```yaml
# Add to the same stack
  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    restart: unless-stopped
    ports:
      - "8191:8191"
    environment:
      LOG_LEVEL: info
```

Then in Prowlarr go to **Settings > Indexers > Add Proxy** and set the FlareSolverr URL to `http://flaresolverr:8191`.

## Monitoring

Use OneUptime to monitor `http://<host>:9696` for HTTP 200. Prowlarr downtime means your *arr applications can no longer search for new releases, so alert immediately on any outage.
