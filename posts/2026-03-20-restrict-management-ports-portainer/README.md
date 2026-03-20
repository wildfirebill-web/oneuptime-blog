# How to Restrict Management Ports in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Ports, Firewall, Hardening

Description: Learn how to restrict access to Portainer's management ports using firewall rules and network configuration to reduce attack surface.

## Portainer's Default Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9000 | HTTP | Web UI (insecure) |
| 9443 | HTTPS | Web UI (secure) |
| 8000 | TCP | Edge Agent communication |
| 9001 | TCP | Portainer Agent (on managed nodes) |

Exposing all these ports to the internet is a significant security risk.

## Best Practice: Close Port 9000 (HTTP)

Always disable the HTTP port and use HTTPS only:

```bash
# Remove the -p 9000:9000 mapping - only expose HTTPS
docker run -d \
  -p 9443:9443 \         # HTTPS only
  -p 8000:8000 \         # Edge agents only if needed
  --name portainer \
  portainer/portainer-ce:latest
```

## Using a Reverse Proxy (Port 443 Instead of 9443)

Route HTTPS traffic through a reverse proxy on standard port 443:

```nginx
# nginx.conf - reverse proxy Portainer on port 443
server {
    listen 443 ssl http2;
    server_name portainer.mycompany.com;

    ssl_certificate /etc/ssl/certs/portainer.crt;
    ssl_certificate_key /etc/ssl/private/portainer.key;

    location / {
        proxy_pass https://portainer:9443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name portainer.mycompany.com;
    return 301 https://$host$request_uri;
}
```

## Firewall Rules with UFW

```bash
# Only allow Portainer from trusted IPs
sudo ufw default deny incoming

# Allow HTTPS only from your office IP range
sudo ufw allow from 203.0.113.0/24 to any port 443 proto tcp

# Allow Edge agent port only from Edge agent servers
sudo ufw allow from 198.51.100.0/28 to any port 8000 proto tcp

# Block the raw Portainer ports externally
sudo ufw deny 9000
sudo ufw deny 9443
sudo ufw deny 9001

sudo ufw enable
```

## Restricting Port 9001 (Agent Port)

The agent port should only accept connections from your Portainer server:

```bash
# On the Portainer Agent host, allow port 9001 only from Portainer server
iptables -A INPUT -p tcp --dport 9001 \
  -s <portainer-server-ip> -j ACCEPT
iptables -A INPUT -p tcp --dport 9001 -j DROP

# Persist with iptables-persistent
apt-get install iptables-persistent
iptables-save > /etc/iptables/rules.v4
```

## Binding to Specific Interfaces

Run Portainer bound to only specific interfaces:

```bash
# Bind to localhost (access via SSH tunnel only)
docker run -d \
  -p 127.0.0.1:9443:9443 \
  --name portainer \
  portainer/portainer-ce:latest

# Access via SSH tunnel
ssh -L 9443:localhost:9443 user@portainer-server
# Then browse: https://localhost:9443
```

## Edge Agent Port Security

```bash
# The Edge agent port (8000) should only be accessible from Edge nodes
# Use a separate firewall group or security group for Edge nodes
# Add Edge node IPs to the allowlist for port 8000
```

## Docker Compose Example with Restricted Ports

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    # Only bind to localhost - let Nginx handle external access
    ports:
      - "127.0.0.1:9443:9443"
    volumes:
      - portainer_data:/data

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "443:443"   # Public HTTPS
      - "80:80"     # Redirect to HTTPS
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/portainer.conf:ro
      - ./certs:/etc/ssl:ro
```

## Conclusion

Restricting Portainer's management ports is a critical security step. Bind to localhost and use a reverse proxy for external access, apply firewall rules to limit source IPs, and never expose the raw HTTP port (9000) in any environment.
