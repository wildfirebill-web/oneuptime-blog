# How to Deploy Transmission via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Transmission, BitTorrent, Downloading, Self-Hosted

Description: Deploy Transmission via Portainer as a lightweight BitTorrent client with a clean web interface and RSS-based auto-downloading capabilities.

## Introduction

Transmission is a lightweight, easy-to-use BitTorrent client known for its minimal resource usage. It's ideal for lower-power devices like NAS boxes and Raspberry Pis where qBittorrent might be too heavy. Deploying via Portainer gives you remote management through its simple web UI.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  transmission:
    image: lscr.io/linuxserver/transmission:latest
    container_name: transmission
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      # Web UI credentials
      - TRANSMISSION_WEB_HOME=/config/transmissionic  # Optional: modern UI
      - USER=admin
      - PASS=change_this_password
    volumes:
      - transmission_config:/config
      # Download directories
      - /mnt/media/downloads:/downloads
      - /mnt/media/downloads/incomplete:/incomplete
    ports:
      - "9091:9091"    # Web UI
      - "51413:51413"  # Peer port
      - "51413:51413/udp"
    restart: unless-stopped

volumes:
  transmission_config:
```

## Access and Configuration

Navigate to `http://<host>:9091` with credentials admin/your_password.

## Custom Settings

The main settings file is `settings.json` in the config volume. Key settings:

```json
{
    "download-dir": "/downloads",
    "incomplete-dir": "/incomplete",
    "incomplete-dir-enabled": true,
    "peer-port": 51413,
    "rpc-authentication-required": true,
    "rpc-username": "admin",
    "rpc-password": "your_password",
    "rpc-whitelist": "127.0.0.1,192.168.*.*",
    "speed-limit-down-enabled": false,
    "speed-limit-up": 100,
    "speed-limit-up-enabled": true,
    "ratio-limit": 1.0,
    "ratio-limit-enabled": true
}
```

## Transmission with OpenVPN

```yaml
version: "3.8"

services:
  transmission-openvpn:
    image: haugene/transmission-openvpn:latest
    container_name: transmission-openvpn
    cap_add:
      - NET_ADMIN
    environment:
      - OPENVPN_PROVIDER=PIA
      - OPENVPN_CONFIG=netherlands
      - OPENVPN_USERNAME=your_vpn_username
      - OPENVPN_PASSWORD=your_vpn_password
      - LOCAL_NETWORK=192.168.1.0/24
      - TRANSMISSION_WEB_UI=combustion   # Or flood-for-transmission
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - transmission_config:/data
      - /mnt/media/downloads:/data/completed
    ports:
      - "9091:9091"
    restart: unless-stopped
```

## Adding Transmission to *arr Stack

Configure in Sonarr/Radarr:

1. **Settings > Download Clients > Add**
2. Select **Transmission**
3. Host: `transmission`, Port: `9091`
4. Username/password from your config
5. Category: `sonarr` or `radarr`

## Resource Usage Comparison

| Feature | Transmission | qBittorrent |
|---------|-------------|-------------|
| Idle RAM | ~30MB | ~100MB |
| UI Complexity | Simple | Feature-rich |
| Web UI | Built-in (basic) | Built-in (advanced) |
| Plugin ecosystem | Minimal | None |
| ARM support | Excellent | Good |

## Conclusion

Transmission deployed via Portainer is ideal for resource-constrained environments like NAS devices and ARM boards where every megabyte of RAM matters. Its simplicity and low resource usage make it reliable as a background download client in automated media pipelines. The VPN-integrated images provide a straightforward path to privacy-protected downloading.
