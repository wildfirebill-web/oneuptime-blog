# How to Fix 'bind() to 0.0.0.0:80 Failed' Errors in Nginx

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, Troubleshooting, IPv4, Port Conflict, Bind error, Linux

Description: Diagnose and resolve Nginx 'bind() to 0.0.0.0:80 failed' errors caused by port conflicts, permission issues, or misconfigured listen directives.

## Introduction

The error `nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)` or `(13: Permission denied)` prevents Nginx from starting. This guide covers all common causes and their fixes.

## Diagnosing the Error

Check the Nginx error log and systemd journal first:

```bash
# Check Nginx error output

sudo nginx -t

# View systemd service logs
sudo journalctl -xeu nginx --no-pager | tail -30

# Check which process is using port 80
sudo ss -tlnp | grep ':80'
# or
sudo lsof -i :80
```

## Cause 1: Another Process Is Using Port 80

The most common cause-Apache, another Nginx instance, or another web server is already listening:

```bash
# Identify the conflicting process
sudo ss -tlnp | grep ':80'
# Output example:
# LISTEN 0 128 0.0.0.0:80  0.0.0.0:*  users:(("apache2",pid=1234,fd=4))

# Option A: Stop the conflicting service
sudo systemctl stop apache2

# Option B: Change it to a different port
sudo sed -i 's/Listen 80/Listen 8080/' /etc/apache2/ports.conf
sudo systemctl restart apache2

# Then start Nginx
sudo systemctl start nginx
```

## Cause 2: Stale Nginx Process Still Running

A crashed or zombie Nginx process may still hold the socket:

```bash
# Check for any running Nginx processes
ps aux | grep nginx

# Kill the master process
sudo kill -QUIT $(cat /var/run/nginx.pid)

# If PID file is gone, kill all nginx processes
sudo pkill nginx

# Remove stale PID file if present
sudo rm -f /var/run/nginx.pid

# Start Nginx fresh
sudo systemctl start nginx
```

## Cause 3: Permission Denied (Port Below 1024)

Non-root processes cannot bind to ports below 1024 without explicit permissions:

```bash
# Check Nginx worker user
grep '^user' /etc/nginx/nginx.conf

# Option A: Use authbind to grant permission to a specific port
sudo apt install authbind
sudo touch /etc/authbind/byport/80
sudo chmod 500 /etc/authbind/byport/80
sudo chown www-data /etc/authbind/byport/80

# Option B: Set CAP_NET_BIND_SERVICE capability
sudo setcap cap_net_bind_service=+ep /usr/sbin/nginx
```

## Cause 4: IPv6 Binding Conflict

If another service is listening on `:::80` (IPv6 wildcard which also covers IPv4 on dual-stack):

```bash
# Check for IPv6 listeners
sudo ss -tlnp | grep ':::80'

# In Nginx, explicitly separate IPv4 and IPv6 listeners
server {
    listen 0.0.0.0:80;   # IPv4 only
    # listen [::]:80;    # Comment out IPv6 if conflicting
}
```

## Cause 5: Duplicate listen Directives in Config

Multiple server blocks listening on the same address:port combination:

```bash
# Find duplicate listen directives
grep -rn 'listen.*:80\|listen 80' /etc/nginx/

# Fix: ensure each address:port is only in one server block,
# or add 'default_server' to exactly one block
server {
    listen 0.0.0.0:80 default_server;
    # ...
}
```

## Cause 6: Nginx Not Fully Stopped Before Restart

```bash
# Graceful reload (preferred-no port binding needed)
sudo nginx -s reload

# Full restart sequence
sudo systemctl stop nginx
sleep 2
sudo systemctl start nginx

# Or use systemctl restart which handles this correctly
sudo systemctl restart nginx
```

## Verifying the Fix

```bash
# Confirm Nginx is listening on port 80
sudo ss -tlnp | grep nginx
# Expected: LISTEN 0 511 0.0.0.0:80 0.0.0.0:*  users:(("nginx",...))

# Test configuration validity
sudo nginx -t
# Expected: nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## Conclusion

"bind() failed" errors almost always fall into one of four categories: port conflict, permission issue, IPv6 collision, or duplicate config. Use `ss -tlnp` to identify what holds the port, resolve the conflict, then validate config with `nginx -t` before restarting. For production servers, use `nginx -s reload` instead of full restarts to avoid downtime during config changes.
