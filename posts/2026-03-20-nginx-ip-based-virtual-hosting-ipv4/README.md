# How to Configure Nginx for IP-Based Virtual Hosting on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, IPv4, Virtual Hosting, Web Server, Linux, Configuration

Description: Learn how to configure Nginx to serve different websites based on the IPv4 address the request arrives on using IP-based virtual hosting.

---

IP-based virtual hosting allows a single Nginx server to serve distinct websites depending on which IPv4 address the client connects to. This is useful when you have multiple public IPs assigned to one server and need each IP to serve different content or apply different policies.

## Prerequisites

Your server must have multiple IPv4 addresses configured. Verify them with:

```bash
# List all IPv4 addresses on the server
ip -4 addr show
```

Example output might show `192.168.1.10` and `192.168.1.20` assigned to the same interface.

## How IP-Based Virtual Hosting Works

Unlike name-based hosting (which uses the HTTP `Host` header), IP-based hosting routes requests solely by the destination IP. Each `server` block `listen`s on a different IP address.

```mermaid
graph TD
    A[Client Request] --> B{Destination IP?}
    B -->|192.168.1.10| C[Site A Server Block]
    B -->|192.168.1.20| D[Site B Server Block]
    C --> E[/var/www/site-a]
    D --> F[/var/www/site-b]
```

## Configuring Nginx Server Blocks

Create separate configuration files for each IP-based virtual host.

```nginx
# /etc/nginx/sites-available/site-a
server {
    # Listen only on the first IPv4 address
    listen 192.168.1.10:80;

    server_name _;  # Catch all hostnames on this IP

    root /var/www/site-a;
    index index.html;

    access_log /var/log/nginx/site-a-access.log;
    error_log  /var/log/nginx/site-a-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```nginx
# /etc/nginx/sites-available/site-b
server {
    # Listen only on the second IPv4 address
    listen 192.168.1.20:80;

    server_name _;

    root /var/www/site-b;
    index index.html;

    access_log /var/log/nginx/site-b-access.log;
    error_log  /var/log/nginx/site-b-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## Enabling the Configuration

```bash
# Symlink the configs into sites-enabled
ln -s /etc/nginx/sites-available/site-a /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/site-b /etc/nginx/sites-enabled/

# Test the configuration for syntax errors
nginx -t

# Reload Nginx to apply changes
systemctl reload nginx
```

## Verifying the Setup

Use `curl` with the `--resolve` flag or specify the IP directly to confirm each virtual host responds correctly.

```bash
# Request the first virtual host by IP
curl http://192.168.1.10/

# Request the second virtual host by IP
curl http://192.168.1.20/
```

## Adding HTTPS per IP

Each IP can have its own TLS certificate by adding a second `server` block for port 443.

```nginx
server {
    listen 192.168.1.10:443 ssl;
    ssl_certificate     /etc/ssl/site-a/fullchain.pem;
    ssl_certificate_key /etc/ssl/site-a/privkey.pem;
    root /var/www/site-a;
    # ... other directives
}
```

## Key Takeaways

- Bind each `server` block to a specific `ip:port` pair instead of `0.0.0.0:80`.
- Use `server_name _` to catch any hostname on that IP.
- IP-based hosting does not rely on the `Host` header, making it suitable for non-HTTP protocols or strict IP segregation policies.
