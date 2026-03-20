# How to Deploy Radarr via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Radarr, Media Management, Docker, Self-Hosting, Movies, Automation

Description: Learn how to deploy Radarr, the automated movie download manager, via Portainer with download client integration and media library configuration.

---

Radarr is the movie counterpart to Sonarr. It monitors release groups, automatically downloads movies in your preferred quality, and moves them into your media library with proper naming. Portainer makes container management simple.

## Prerequisites

- Portainer running
- A download client (qBittorrent or Transmission)
- A dedicated directory for your movie library

## Compose Stack

Use the same parent mount path trick as Sonarr so Radarr can hardlink completed downloads into the movies directory without copying:

```yaml
version: "3.8"

services:
  radarr:
    image: linuxserver/radarr:latest
    restart: unless-stopped
    ports:
      - "7878:7878"
    environment:
      PUID: 1000    # Must match ownership of your media files
      PGID: 1000
      TZ: America/New_York
    volumes:
      - radarr_config:/config
      - /mnt/data/downloads:/downloads    # Where download client puts files
      - /mnt/data/media/movies:/movies    # Final movie library location

volumes:
  radarr_config:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `radarr`.
3. Adjust `PUID`/`PGID` and host paths.
4. Click **Deploy the stack**.

Open `http://<host>:7878`.

## Connecting Prowlarr as Indexer Manager

If you have Prowlarr running, sync all your indexers to Radarr automatically:

1. In Prowlarr go to **Settings > Apps > Add Application**.
2. Choose Radarr.
3. Set **Radarr Server** to `http://radarr:7878` (using the container name if on the same Docker network).
4. Paste the **API Key** from Radarr's **Settings > General**.
5. Click **Test** then **Save**.

All Prowlarr indexers will immediately appear in Radarr.

## Quality Profiles

Radarr's quality profiles let you specify preferred resolutions and codecs. For a home server a typical setup is:

- **4K** profile: prefer 2160p Remux or 2160p BluRay
- **HD** profile: prefer 1080p BluRay or 1080p WEB-DL

## Monitoring

Monitor `http://<host>:7878` with OneUptime for HTTP 200. For deeper health checking, poll `/api/v3/health` with an `X-Api-Key` header. An empty JSON array means no health warnings; any entries indicate issues needing attention.
