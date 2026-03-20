# How to Deploy qBittorrent via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, qBittorrent, BitTorrent, Downloading, Self-Hosted

Description: Deploy qBittorrent via Portainer with a web UI for remote torrent management, optional VPN kill switch, and proper file permission handling.

## Introduction

qBittorrent is a feature-rich, open-source BitTorrent client with a web UI. When deployed via Portainer, it provides remote torrent management accessible from any browser. Combined with Sonarr and Radarr, it forms the download client in an automated media pipeline.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - WEBUI_PORT=8080
      # qBittorrent version for specific releases
      - DOCKER_MODS=linuxserver/mods:qbittorrent-vuetorrent   # Optional: Modern UI
    volumes:
      - qbittorrent_config:/config
      # Download directory
      - /mnt/media/downloads:/downloads
    ports:
      - "8080:8080"    # Web UI
      - "6881:6881"    # BitTorrent port
      - "6881:6881/udp"
    restart: unless-stopped

volumes:
  qbittorrent_config:
```

## Initial Access

Navigate to `http://<host>:8080`. Default credentials:
- Username: `admin`
- Password: `adminadmin` (change this immediately!)

## Configuring for *arr Integration

In qBittorrent Web UI:

1. **Options > Web UI**: Change password
2. **Options > Downloads**: Set default save path to `/downloads`
3. **Options > BitTorrent**: Configure port (6881 recommended)

For Sonarr/Radarr integration, the download category is important:

- In Sonarr: Create category `sonarr` in qBittorrent settings
- In Radarr: Create category `radarr`

qBittorrent separates downloads into category subdirectories:
- `/downloads/sonarr/` — TV show downloads
- `/downloads/radarr/` — Movie downloads

## VPN Kill Switch Configuration

For privacy, use a VPN-enabled image:

```yaml
version: "3.8"

services:
  qbittorrent:
    image: binhex/arch-qbittorrentvpn:latest
    container_name: qbittorrent-vpn
    cap_add:
      - NET_ADMIN
    environment:
      - VPN_ENABLED=yes
      - VPN_USER=your_vpn_username
      - VPN_PASS=your_vpn_password
      - VPN_PROV=pia           # Private Internet Access
      - VPN_CLIENT=wireguard   # or openvpn
      - VPN_REGION=Netherlands
      - WEBUI_PORT=8080
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - qbittorrent_config:/config
      - /mnt/media/downloads:/downloads
    ports:
      - "8080:8080"
    restart: unless-stopped
```

## Download Categories and Paths

Configure in qBittorrent **Options > Downloads**:

```
Default save path: /downloads
Keep incomplete torrents in: /downloads/incomplete

Category mappings:
sonarr  → /downloads/tvshows/
radarr  → /downloads/movies/
```

## Automating Cleanup

After downloads complete and Sonarr/Radarr import them, configure automatic removal:

**Options > BitTorrent:**
- Seeding ratio limit: 1.0
- Delete torrent and data when seeding complete: Yes (or seed and then remove)

## Conclusion

qBittorrent deployed via Portainer provides reliable, web-accessible torrent downloading for your media automation pipeline. The category system integrates cleanly with Sonarr and Radarr for automatic file management after downloads complete. The optional VPN kill switch integration ensures downloading activity is protected by your VPN.
