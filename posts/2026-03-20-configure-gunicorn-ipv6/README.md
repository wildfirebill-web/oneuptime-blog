# How to Configure Gunicorn for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Gunicorn, Python, IPv6, WSGI, Deployment, Dual-Stack, Workers

Description: Configure Gunicorn to listen on IPv6 addresses, bind to multiple interfaces, and integrate with NGINX for dual-stack Python application deployment.

## Introduction

Gunicorn (Green Unicorn) is the most popular WSGI server for Python applications. It supports IPv6 binding using bracket notation in the bind address. This post covers single and dual-stack binding, worker configuration, and NGINX integration.

## Step 1: Basic IPv6 Binding

```bash
# Bind to all IPv6 interfaces (dual-stack on Linux)
gunicorn --bind "[::]:8000" app:application

# Bind to IPv6 loopback only
gunicorn --bind "[::1]:8000" app:application

# Bind to specific IPv6 address
gunicorn --bind "[2001:db8::1]:8000" app:application

# Dual-stack: bind to both IPv4 and IPv6
gunicorn \
    --bind "0.0.0.0:8000" \
    --bind "[::]:8000" \
    app:application
```

## Step 2: Gunicorn Configuration File

```python
# gunicorn.conf.py

# IPv6 binding
bind = [
    "[::]:8000",      # All IPv6 interfaces
    "0.0.0.0:8000",   # All IPv4 interfaces
]

# Worker configuration
workers = 4
worker_class = "sync"   # or "gevent", "uvicorn.workers.UvicornWorker"
threads = 2
worker_connections = 1000

# Timeouts
timeout = 30
keepalive = 5

# Access logging
accesslog = "/var/log/gunicorn/access.log"
errorlog  = "/var/log/gunicorn/error.log"
loglevel  = "info"

# Log client IPv6 addresses
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s"'
# %(h)s = client IP — shows the IPv6 address

# Process naming
proc_name = "myapp"

# Security
forwarded_allow_ips = "::1,127.0.0.1,2001:db8::1"
```

## Step 3: Forwarded Allow IPs for IPv6 Proxy

```python
# gunicorn.conf.py

# Trust these IPv6 addresses for X-Forwarded-For
# (Gunicorn will use the header's IP as client address for logging)
forwarded_allow_ips = "::1,127.0.0.1"

# Or trust a subnet
forwarded_allow_ips = "::1,127.0.0.0/8,2001:db8::/32"

# Trust all (dangerous — only in trusted networks)
# forwarded_allow_ips = "*"
```

## Step 4: Systemd Service

```ini
# /etc/systemd/system/gunicorn.service
[Unit]
Description=Gunicorn IPv6 WSGI server
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/myapp
Environment="PATH=/var/www/myapp/venv/bin"
ExecStart=/var/www/myapp/venv/bin/gunicorn \
    --config /var/www/myapp/gunicorn.conf.py \
    myapp.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now gunicorn
systemctl status gunicorn

# Verify IPv6 listening
ss -lntp | grep :8000
# tcp  LISTEN  0  2048  [::]:8000  [::]:*  users:(("gunicorn",...))
```

## Step 5: NGINX + Gunicorn IPv6

```nginx
upstream gunicorn_backend {
    server [::1]:8000;
}

server {
    listen [::]:80;
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://gunicorn_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /var/www/myapp/static/;
    }
}
```

## Step 6: Async Workers (Uvicorn) for ASGI

```bash
# For FastAPI / Django ASGI / Starlette
gunicorn myapp.asgi:application \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind "[::]:8000" \
    --workers 4
```

## Troubleshooting

```bash
# Error: Invalid address '[::]:8000'
# Fix: Ensure brackets are included
# Wrong:  --bind ":::8000"
# Right:  --bind "[::]:8000"

# Error: Address already in use
# Check what's on port 8000
ss -lntp | grep :8000
fuser 8000/tcp

# Verify Gunicorn actually bound to IPv6
curl -6 http://[::1]:8000/health
```

## Conclusion

Gunicorn binds to IPv6 with bracket notation: `--bind "[::]:8000"`. Use `gunicorn.conf.py` for production configuration including `forwarded_allow_ips` for IPv6 proxy trust. Run Gunicorn behind NGINX with `proxy_pass http://[::1]:8000` for best performance. Monitor Gunicorn worker health and response times with OneUptime.
