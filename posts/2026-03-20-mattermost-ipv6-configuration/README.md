# How to Configure Mattermost with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mattermost, IPv6, Team Chat, Messaging, Linux, Self-Hosted

Description: Configure Mattermost team messaging platform to accept connections from IPv6 clients, including server configuration, reverse proxy setup, and WebSocket support.

---

Mattermost is a self-hosted team messaging platform. Enabling IPv6 access involves configuring the Mattermost server's listener address and the reverse proxy (Nginx) to accept IPv6 connections.

## Configuring Mattermost for IPv6

```json
// /opt/mattermost/config/config.json

{
  "ServiceSettings": {
    "SiteURL": "https://mattermost.example.com",
    "ListenAddress": ":8065",
    "ConnectionSecurity": "",
    "EnableAPIv3": false,
    "EnableAPITeamDeletion": false,

    "WebsocketURL": "wss://mattermost.example.com",

    "ReadTimeout": 300,
    "WriteTimeout": 300,

    "MaximumLoginAttempts": 10
  }
}
```

```text
Note: Mattermost's "ListenAddress": ":8065" binds to all interfaces
including IPv6 (equivalent to [::]:8065 in Go)

Go's net package listens on all IPv4 and IPv6 interfaces
when binding to ":port" on dual-stack systems.
```

## Nginx Reverse Proxy for Mattermost IPv6

```nginx
# /etc/nginx/sites-available/mattermost

upstream mattermost {
    server 127.0.0.1:8065;
    keepalive 32;
}

server {
    listen 80;
    listen [::]:80;
    server_name mattermost.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name mattermost.example.com;

    ssl_certificate /etc/ssl/certs/mattermost.crt;
    ssl_certificate_key /etc/ssl/private/mattermost.key;

    location / {
        proxy_pass http://mattermost;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Frame-Options SAMEORIGIN;
        proxy_buffers 256 16k;
        proxy_buffer_size 16k;
        proxy_http_version 1.1;
        proxy_read_timeout 600;
        proxy_connect_timeout 300;
        proxy_set_header Connection "";
        client_max_body_size 50m;
    }

    # WebSocket support for real-time messaging
    location ~ /api/v[0-9]+/(users/)?websocket$ {
        proxy_pass http://mattermost;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Frame-Options SAMEORIGIN;
        proxy_buffers 256 16k;
        proxy_buffer_size 16k;
        client_max_body_size 50m;
        proxy_read_timeout 600;
    }
}
```

## Mattermost Database over IPv6

```json
// /opt/mattermost/config/config.json

{
  "SqlSettings": {
    "DriverName": "postgres",
    "DataSource": "postgres://mattermost:password@[2001:db8::postgres]:5432/mattermost?sslmode=disable",
    "MaxIdleConns": 10,
    "MaxOpenConns": 300,
    "QueryTimeout": 30
  }
}
```

## Firewall for Mattermost IPv6

```bash
# Allow Mattermost via reverse proxy
sudo ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT

# If accessing Mattermost directly (no proxy)
sudo ip6tables -A INPUT -p tcp --dport 8065 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Start and Verify Mattermost

```bash
# Restart Mattermost
sudo systemctl restart mattermost
sudo systemctl enable mattermost

# Check Mattermost is listening on IPv6
ss -6 -tlnp | grep 8065

# Reload Nginx
sudo systemctl reload nginx

# Test access over IPv6
curl -6 -I https://mattermost.example.com/

# Test API
curl -6 https://mattermost.example.com/api/v4/system/ping

# Check logs for IPv6 connections
sudo journalctl -u mattermost -f | grep "2001:"
```

Mattermost's Go-based server naturally listens on both IPv4 and IPv6 when bound to `:8065`, making the IPv6 listener active by default on dual-stack systems, with Nginx's `listen [::]:443` being the only additional configuration needed for IPv6 access.
