# How to Deploy Emby via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Emby, Media Server, Self-Hosted, Streaming

Description: Deploy Emby Media Server via Portainer for a polished media streaming experience with a clean interface, live TV support, and optional Emby Premiere features.

## Introduction

Emby is a media server that sits between Plex and Jellyfin in terms of open-source-ness. The server is source-available and free to use, with an optional Emby Premiere subscription for advanced features. Deploying via Portainer provides easy management and updates.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  emby:
    image: emby/embyserver:latest
    container_name: emby
    network_mode: host   # Required for DLNA and network discovery
    environment:
      UID: 1000
      GID: 1000
      GIDLIST: 44    # video group for GPU access
    volumes:
      # Emby configuration and database
      - emby_config:/config
      # Media directories
      - /mnt/media/movies:/movies:ro
      - /mnt/media/tvshows:/tvshows:ro
      - /mnt/media/music:/music:ro
    # Uncomment for Intel hardware transcoding
    # devices:
    #   - /dev/dri:/dev/dri
    restart: unless-stopped

volumes:
  emby_config:
```

## Access and Setup

Navigate to `http://<host>:8096` and complete the setup wizard.

## Hardware Transcoding

### Intel iGPU

```yaml
services:
  emby:
    devices:
      - /dev/dri:/dev/dri
    environment:
      GIDLIST: 44,109   # video and render groups
```

Enable in Emby: **Settings > Transcoding > Hardware acceleration > Intel QuickSync Video**

### NVIDIA

```yaml
services:
  emby:
    runtime: nvidia
    environment:
      NVIDIA_VISIBLE_DEVICES=all
      NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
```

## Emby Premiere Features

With Emby Premiere subscription:
- Hardware transcoding on all platforms
- Mobile sync (offline downloads)
- Cover art provider
- Premium themes

Configure at **Settings > Emby Premiere**

## Live TV with HDHomeRun

```yaml
services:
  emby:
    # HDHomeRun tuner is accessed over network, no special config needed
    network_mode: host
```

In Emby: **Settings > Live TV > Add tuner device > HDHomeRun**

## Emby with Traefik

```yaml
services:
  emby:
    network_mode: bridge  # Can't use host mode with Traefik
    ports:
      - "8096:8096"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.emby.rule=Host(`media.example.com`)"
      - "traefik.http.routers.emby.entrypoints=websecure"
      - "traefik.http.routers.emby.tls.certresolver=letsencrypt"
      - "traefik.http.services.emby.loadbalancer.server.port=8096"
```

Note: Using bridge mode instead of host mode disables DLNA discovery.

## Conclusion

Emby deployed via Portainer provides a polished media streaming experience with a user-friendly interface. Its Live TV integration and Premiere features appeal to users who want a more managed experience than Jellyfin provides. The linuxserver.io image offers good container management with proper UID/GID handling for file permissions.
