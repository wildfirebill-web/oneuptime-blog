# How to Set Up Grafana IP-Based Access Control with IPv4 Ranges

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Grafana, IPv4, Access Control, Security, Configuration, Monitoring, Reverse Proxy

Description: Learn how to restrict Grafana access to specific IPv4 addresses and ranges using Grafana's configuration settings and a reverse proxy.

---

Restricting Grafana access to specific IPv4 addresses prevents unauthorized users from reaching your monitoring dashboards. This can be done at the Grafana configuration level or via a reverse proxy layer.

## Method 1: Grafana HTTP Binding

Bind Grafana to a specific IPv4 address to prevent access from other interfaces.

```ini
# /etc/grafana/grafana.ini

[server]
# Bind Grafana to a specific IPv4 address (e.g., internal network only)
# 0.0.0.0 = all interfaces; specify an IP to restrict
http_addr = 192.168.1.10

http_port = 3000

# Domain for the Grafana URL (used in email links)
domain = grafana.example.com
```

```bash
systemctl restart grafana-server
ss -tlnp | grep :3000   # Should show only 192.168.1.10:3000
```

## Method 2: Nginx Reverse Proxy with IP Restriction

Run Grafana on localhost only and expose it through Nginx with IP-based allow/deny.

```ini
# /etc/grafana/grafana.ini
[server]
http_addr = 127.0.0.1   # Only listen on loopback
http_port = 3000
```

```nginx
# /etc/nginx/sites-available/grafana
server {
    listen 192.168.1.10:80;
    server_name grafana.example.com;

    # Allow access from internal networks only
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;

    # Allow specific admin workstations
    allow 203.0.113.5;

    # Deny all other IPs
    deny all;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
nginx -t && systemctl reload nginx
```

## Method 3: Grafana IP Range Feature (Enterprise)

Grafana Enterprise supports IP range restrictions via the `authorized_ip_ranges` setting.

```ini
# /etc/grafana/grafana.ini (Grafana Enterprise)
[auth]
# Restrict login to these IPv4 ranges
authorized_ip_ranges = 192.168.1.0/24, 10.0.0.0/8
```

## Method 4: Firewall Rules

For all Grafana editions, use OS-level firewall rules.

```bash
# Allow Grafana access from the internal network
ufw allow from 192.168.1.0/24 to any port 3000

# Allow from a specific admin machine
ufw allow from 203.0.113.5 to any port 3000

# Deny all other access to port 3000
ufw deny 3000

# or with iptables:
iptables -A INPUT -p tcp --dport 3000 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 3000 -j DROP
```

## Logging Access Denials

```bash
# If using Nginx, denied IPs appear in the error log
tail -f /var/log/nginx/error.log | grep "forbidden"

# Or enable access logging to see both allowed and denied
tail -f /var/log/nginx/access.log | awk '$9 == "403"'
```

## Key Takeaways

- Set `http_addr` in `grafana.ini` to bind Grafana to an internal IPv4 address only.
- Use Nginx `allow`/`deny` rules in the reverse proxy for granular IP-based access control.
- Firewall rules (ufw/iptables) provide the deepest protection independent of Grafana or Nginx.
- Always pair IP restrictions with authentication (Grafana login) for defense in depth.
