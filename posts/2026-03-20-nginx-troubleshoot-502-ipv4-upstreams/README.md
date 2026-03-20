# How to Troubleshoot Nginx 502 Bad Gateway with IPv4 Upstreams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, 502, Troubleshooting, IPv4, Upstream, Bad Gateway

Description: Systematically diagnose and fix Nginx 502 Bad Gateway errors when proxying to IPv4 upstream servers by checking connectivity, configuration, and backend health.

## Introduction

A 502 Bad Gateway from Nginx means it received an invalid or no response from the upstream server. This could be the upstream not running, unreachable, returning garbage, or the connection being refused.

## Step 1: Read the Nginx Error Log

The error log always reveals the specific cause:

```bash
sudo tail -f /var/log/nginx/error.log

# Common 502 messages and their meanings:
# connect() failed (111: Connection refused) → backend not running on that port
# connect() failed (110: Connection timed out) → backend unreachable/firewalled
# upstream prematurely closed connection → backend crashed mid-response
# recv() failed (104: Connection reset by peer) → backend reset the connection
# no live upstreams while connecting to upstream → all backends failed health check
```

## Step 2: Test Backend Connectivity Directly

Bypass Nginx to test if the backend itself is responsive:

```bash
# Test if the upstream port is open
curl -v http://192.168.1.10:8080/health

# Or use nc for a raw TCP test
nc -zv 192.168.1.10 8080
# Expected: succeeded!

# Check the backend process is listening
ssh 192.168.1.10 "ss -tlnp | grep 8080"
```

## Step 3: Verify Nginx Upstream Configuration

```bash
# Print the effective upstream configuration
nginx -T | grep -A 20 'upstream'

# Common mistakes:
# Wrong port in upstream block
# Typo in IP address
# Backend listening on 127.0.0.1 (not accessible remotely)
```

Ensure backends listen on the correct interface:

```bash
# On the backend server, verify it's not binding to loopback only
ss -tlnp | grep 8080
# BAD:  LISTEN 0 128 127.0.0.1:8080  — only accessible locally
# GOOD: LISTEN 0 128 0.0.0.0:8080   — accessible on all interfaces
```

## Step 4: Check Firewall Rules

```bash
# On the Nginx server: can it reach the backend?
telnet 192.168.1.10 8080

# Check iptables on the backend for blocking rules
ssh 192.168.1.10 "sudo iptables -L INPUT -n | grep 8080"

# If blocked, add an allow rule
ssh 192.168.1.10 "sudo iptables -I INPUT -s 192.168.1.1 -p tcp --dport 8080 -j ACCEPT"
```

## Step 5: Increase Upstream Timeouts

If backends are slow and Nginx is timing out before getting a response:

```nginx
# /etc/nginx/conf.d/upstream.conf

upstream app_backends {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;

    location / {
        proxy_pass http://app_backends;

        # Increase timeouts if backends are slow to respond
        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        # Try the next upstream on failure
        proxy_next_upstream error timeout http_502 http_503;
        proxy_next_upstream_tries 3;
    }
}
```

## Step 6: Check for Unix Socket Issues

If using Unix sockets instead of TCP:

```bash
# Verify the socket file exists and Nginx can access it
ls -la /var/run/app.sock

# Check permissions (Nginx worker user must be able to connect)
stat /var/run/app.sock

# Fix permissions
chmod 660 /var/run/app.sock
chown root:www-data /var/run/app.sock
```

## Step 7: Enable Upstream Debug Logging

```nginx
# /etc/nginx/nginx.conf
error_log /var/log/nginx/error.log debug;
```

```bash
# Filter upstream-specific debug messages
tail -f /var/log/nginx/error.log | grep upstream
```

## Quick Diagnostic Script

```bash
#!/bin/bash
# Check all upstreams from Nginx config
BACKEND_IPS=("192.168.1.10:8080" "192.168.1.11:8080")

for backend in "${BACKEND_IPS[@]}"; do
    ip="${backend%:*}"
    port="${backend#*:}"
    if nc -z -w 3 "$ip" "$port" 2>/dev/null; then
        echo "UP:   $backend"
    else
        echo "DOWN: $backend"
    fi
done
```

## Conclusion

Nginx 502 errors follow a consistent diagnostic path: check the error log for the specific message, test backend connectivity directly, verify configuration, check firewalls, and tune timeouts. The most common cause is a backend that has stopped running or is bound to loopback only. Enable debug logging temporarily for complex multi-backend scenarios.
