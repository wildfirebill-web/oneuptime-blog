# How to Use the Lissy93 Community Templates Collection with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Templates, Community, DevOps

Description: Learn how to use the popular Lissy93 community template collection to expand your Portainer application catalog with hundreds of self-hosted apps.

## Introduction

The Lissy93 community templates collection (also known as the Portainer Templates project) is a community-maintained catalog of Docker applications ready to deploy via Portainer. It includes hundreds of self-hosted applications - from media servers and home automation to development tools and productivity apps. This guide shows you how to add it to your Portainer instance.

## Prerequisites

- Portainer CE or BE installed
- Admin access to Portainer settings
- Internet access from the Portainer host (to fetch template JSON and pull images)

## About the Lissy93 Templates Collection

The collection is hosted at:
- GitHub: [github.com/Lissy93/portainer-templates](https://github.com/Lissy93/portainer-templates)
- Raw JSON URL: `https://raw.githubusercontent.com/Lissy93/portainer-templates/main/templates.json`

It includes applications in categories such as:

- **Media**: Plex, Jellyfin, Emby, Navidrome
- **Downloads**: qBittorrent, Transmission, Sonarr, Radarr
- **Development**: Gitea, Drone CI, code-server, Gitlab
- **Productivity**: Nextcloud, OnlyOffice, Paperless
- **Monitoring**: Prometheus, Grafana, Uptime Kuma, NetData
- **Networking**: Pi-hole, Nginx Proxy Manager, WireGuard
- **Home Automation**: Home Assistant, Node-RED
- **Security**: Vaultwarden, Authentik, Keycloak

## Step 1: Configure the Templates URL

1. Log in to Portainer as an administrator
2. Click **Settings** in the left sidebar
3. Find the **App Templates** section
4. Replace the current URL with:

```text
https://raw.githubusercontent.com/Lissy93/portainer-templates/main/templates.json
```

5. Click **Save settings**

## Step 2: Browse the New Templates

1. Select your Docker environment
2. Click **App Templates** in the sidebar
3. You now see the community template catalog with hundreds of applications
4. Use the search bar to find specific apps

## Step 3: Deploy an Application from the Collection

### Example: Deploy Uptime Kuma

1. Search for "Uptime Kuma" in the templates
2. Click on the Uptime Kuma template
3. Configure:

```text
Container name: uptime-kuma
Port:          3001 → 3001
Volume:        /data/uptime-kuma → /app/data
```

4. Click **Deploy the container**
5. Access Uptime Kuma at `http://your-host:3001`

### Example: Deploy Vaultwarden (Bitwarden-compatible)

1. Search for "Vaultwarden"
2. Configure the stack template:

```text
Stack name:    vaultwarden
Port:          80 → 80
Data volume:   /data/vaultwarden → /data
Admin token:   [generate-secure-token]
```

3. Deploy and access the web vault

### Example: Deploy Pi-hole

1. Search for "Pi-hole"
2. Configure:

```text
Container name:    pihole
Web password:      [your-admin-password]
DNS port:          53 → 53/udp
Web UI port:       8080 → 80
```

## Step 4: Customize for Your Environment

Many community templates use standard environment variable patterns. Override them as needed:

```bash
# Common customizations

TZ=America/New_York          # Set your timezone
PUID=1000                    # User ID for file permissions
PGID=1000                    # Group ID for file permissions
```

## Step 5: Combine with Your Own Templates

To use both the Lissy93 collection AND your own templates, you need to merge the JSON files:

```bash
# Download the community templates
curl -s https://raw.githubusercontent.com/Lissy93/portainer-templates/main/templates.json \
  -o /tmp/community-templates.json

# Create a merge script
python3 << 'EOF'
import json

# Load community templates
with open('/tmp/community-templates.json') as f:
    community = json.load(f)

# Your custom templates
custom = {
  "version": "2",
  "templates": [
    {
      "type": 1,
      "title": "My Internal App",
      "description": "Internal company application",
      "image": "registry.company.com/myapp:latest",
      "categories": ["internal"],
      "platform": "linux",
      "restart_policy": "unless-stopped"
    }
  ]
}

# Merge: community templates + custom templates
merged = {
    "version": "2",
    "templates": community["templates"] + custom["templates"]
}

with open('/opt/portainer-templates/templates.json', 'w') as f:
    json.dump(merged, f, indent=2)

print(f"Merged {len(community['templates'])} community + {len(custom['templates'])} custom templates")
EOF
```

Host the merged file on your web server and configure Portainer to use it.

## Staying Updated

The community collection is regularly updated with new templates and fixes. To get updates:

```bash
# Re-download and re-merge periodically
# Set up a cron job:
0 2 * * 0 /opt/portainer-templates/update-merged-templates.sh
```

## Important Notes

- Community templates pull images from Docker Hub and other public registries
- Review each template's Compose file before deploying in production
- Not all applications in the collection are suitable for production use without additional hardening
- Some templates may be outdated; always verify image tags are current

## Finding Template Categories

```bash
# Explore available categories in the collection
curl -s https://raw.githubusercontent.com/Lissy93/portainer-templates/main/templates.json | \
  python3 -c "import json,sys; data=json.load(sys.stdin); \
  cats = set(c for t in data['templates'] for c in t.get('categories',[])); \
  print('\n'.join(sorted(cats)))"
```

## Conclusion

The Lissy93 community templates collection dramatically expands your Portainer template catalog with hundreds of popular self-hosted applications. It is perfect for home labs, small teams, and anyone exploring self-hosting. Configure the URL in Portainer settings and start exploring the catalog. For production environments, review templates carefully and combine with your own curated internal templates.
