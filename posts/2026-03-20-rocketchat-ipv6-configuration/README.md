# How to Configure Rocket.Chat with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rocket.Chat, IPv6, Team Chat, Messaging, Nginx, Node.js, Self-Hosted

Description: Configure Rocket.Chat team messaging platform to serve users connecting from IPv6 networks by updating Node.js server binding and Nginx reverse proxy configuration.

---

Rocket.Chat is an open-source team messaging platform built on Node.js. Its server binds to all interfaces by default, and enabling IPv6 access primarily involves ensuring the Nginx reverse proxy listens on IPv6 addresses.

## Rocket.Chat Server IPv6 Binding

```bash
# Rocket.Chat uses Node.js which binds to all interfaces by default
# Check current binding
ss -tlnp | grep 3000

# The Node.js HTTP server listens on 0.0.0.0:3000 by default
# This also covers IPv6 via dual-stack socket on most Linux systems

# To explicitly configure, set environment variable
# /opt/Rocket.Chat/environment
PORT=3000
ROOT_URL=https://chat.example.com
MONGO_URL=mongodb://localhost:27017/rocketchat
MONGO_OPLOG_URL=mongodb://localhost:27017/local
```

## Nginx Reverse Proxy for Rocket.Chat IPv6

```nginx
# /etc/nginx/sites-available/rocketchat
upstream backend {
    server 127.0.0.1:3000;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    listen [::]:80;
    server_name chat.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name chat.example.com;

    ssl_certificate /etc/ssl/certs/rocketchat.crt;
    ssl_certificate_key /etc/ssl/private/rocketchat.key;

    # Main location
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-NginX-Proxy true;
        proxy_redirect off;
        proxy_read_timeout 1200;
        proxy_send_timeout 1200;
    }
}
```

```bash
# Enable site and reload Nginx
sudo ln -s /etc/nginx/sites-available/rocketchat /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# Verify Nginx listens on IPv6
ss -6 -tlnp | grep nginx
```

## MongoDB over IPv6 for Rocket.Chat

```bash
# /etc/mongod.conf - Enable MongoDB on IPv6
net:
  port: 27017
  bindIp: 127.0.0.1,::1  # Both IPv4 and IPv6 loopback
  # For remote access over IPv6:
  # bindIp: 0.0.0.0,::

sudo systemctl restart mongod

# Verify MongoDB on IPv6
ss -6 -tlnp | grep 27017

# Rocket.Chat connection string for IPv6 MongoDB
# MONGO_URL=mongodb://[::1]:27017/rocketchat
```

## Rocket.Chat Admin Configuration

```
After accessing Rocket.Chat over IPv6:

1. Admin > General > Site URL: https://chat.example.com
   (Use FQDN, not raw IPv6 address)

2. Admin > Email > SMTP Settings:
   - If SMTP server is IPv6:
   - Host: smtp.example.com (with AAAA record)

3. Push Notifications:
   - Works regardless of server IPv6 setting
   - Uses push gateway over standard HTTPS
```

## Firewall for Rocket.Chat IPv6

```bash
# Allow HTTPS via Nginx
sudo ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT

# Allow direct access to Rocket.Chat (if not using proxy)
sudo ip6tables -A INPUT -p tcp --dport 3000 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing Rocket.Chat IPv6 Access

```bash
# Test web interface over IPv6
curl -6 -I https://chat.example.com/

# Test REST API over IPv6
curl -6 https://chat.example.com/api/v1/info

# Login via API over IPv6
curl -6 -X POST \
  https://chat.example.com/api/v1/login \
  -H 'Content-Type: application/json' \
  -d '{"user":"admin","password":"password"}'

# Test WebSocket (for real-time messaging)
# WebSocket uses same HTTPS port

# Check Nginx access log for IPv6 clients
sudo tail -f /var/log/nginx/access.log | grep "2001:\|::1"
```

Rocket.Chat's Node.js server combined with Nginx's `listen [::]:443` configuration provides full IPv6 access to team messaging, with WebSocket connections for real-time messaging working over IPv6 through the same reverse proxy configuration as standard HTTP requests.
