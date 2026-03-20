# How to Deploy Plex Media Server via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Plex, Media Server, Self-Hosted, Streaming

Description: Deploy Plex Media Server via Portainer with access to your local media library, optional hardware transcoding, and remote access for streaming anywhere.

## Introduction

Plex Media Server organizes and streams your personal media library — movies, TV shows, music, and photos — to any device. Deploying via Portainer gives you easy updates and clear volume management for your media directories.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  plex:
    image: plexinc/pms-docker:latest
    container_name: plex
    network_mode: host   # Required for local network discovery (GDM)
    environment:
      TZ: America/New_York
      # Get claim token from plex.tv/claim
      PLEX_CLAIM: "claim-xxxxxxxxxxxxxxxxxxxx"
      ADVERTISE_IP: http://192.168.1.100:32400/
    volumes:
      # Plex configuration and database
      - plex_config:/config
      # Transcode temp directory (use RAM disk for performance)
      - plex_transcode:/transcode
      # Media directories (bind mount to your actual media)
      - /mnt/media/movies:/movies:ro
      - /mnt/media/tvshows:/tvshows:ro
      - /mnt/media/music:/music:ro
      - /mnt/media/photos:/photos:ro
    restart: unless-stopped

volumes:
  plex_config:
  plex_transcode:
```

## Getting a Claim Token

1. Go to `plex.tv/claim` while logged in to your Plex account
2. Copy the claim token (valid for 4 minutes)
3. Set it as `PLEX_CLAIM` in the environment

## Hardware Transcoding

For Intel Quick Sync or NVIDIA GPU transcoding:

### Intel (iGPU)

```yaml
services:
  plex:
    devices:
      - /dev/dri:/dev/dri   # Intel hardware transcoding
    environment:
      - PLEX_CLAIM=claim-xxxx
```

### NVIDIA GPU

```yaml
services:
  plex:
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
      - PLEX_CLAIM=claim-xxxx
```

## Accessing Plex

- Local: `http://<host>:32400/web`
- Enable remote access in **Settings > Remote Access**

## Optimizing Plex Performance

```yaml
environment:
  # Use RAM-based transcode directory for better performance
  PLEX_MEDIA_SERVER_TRANSCODER_TEMP_DIRECTORY: /dev/shm/plex-transcode
```

For better transcode performance, use a tmpfs mount:

```yaml
volumes:
  - type: tmpfs
    target: /transcode
    tmpfs:
      size: 4294967296  # 4GB RAM for transcoding
```

## Plex Pass Features

With Plex Pass, you can enable:
- Hardware transcoding (via **Settings > Transcoder**)
- Live TV & DVR
- Mobile sync

## Conclusion

Plex Media Server deployed via Portainer provides a polished media streaming experience for your entire media library. Host network mode ensures Plex discovery protocols work correctly on your local network. Persistent volumes separate configuration from media, making it safe to update Plex without risking your library metadata or settings.
