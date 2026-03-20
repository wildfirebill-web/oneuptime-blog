# How to Set Up Nginx Round-Robin Load Balancing Across IPv4 Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, Round Robin, Load Balancing, IPv4, HTTP, Servers

Description: Configure Nginx to distribute HTTP requests across multiple IPv4 backend servers using the default round-robin algorithm and verify even traffic distribution.

## Introduction

Nginx's default load balancing algorithm is round robin — requests are distributed to backend servers in sequential order. Each server receives requests one at a time in rotation. This is ideal for stateless applications where all backend servers are identical in capability.

## Configuration

```nginx
# /etc/nginx/conf.d/roundrobin.conf

upstream backend-pool {
    # Round robin is the default — no directive needed
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
    server 10.0.1.12:8080;
}

server {
    listen 80;
    server_name lb.example.com;

    location / {
        proxy_pass http://backend-pool;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout  5s;
        proxy_send_timeout     30s;
        proxy_read_timeout     30s;
    }
}
```

## How Round Robin Works

```
Request 1 → 10.0.1.10
Request 2 → 10.0.1.11
Request 3 → 10.0.1.12
Request 4 → 10.0.1.10  (cycle repeats)
Request 5 → 10.0.1.11
...
```

## Verifying Distribution

Each backend should add a response header or body identifying itself:

```bash
# Send 9 requests and record which server responds
for i in $(seq 1 9); do
  curl -s -H "Host: lb.example.com" http://localhost/ | grep "server-id"
done
```

Expected output (3 servers):
```
server-id: 10.0.1.10
server-id: 10.0.1.11
server-id: 10.0.1.12
server-id: 10.0.1.10
... (repeating)
```

## Using Passive Health Checks with Round Robin

```nginx
upstream backend-pool {
    server 10.0.1.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=30s;
}
```

If `10.0.1.11` fails 3 times, Nginx stops sending to it for 30 seconds.

## Error Handling with next_upstream

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://backend-pool;

        # Retry on these errors
        proxy_next_upstream error timeout http_500 http_502 http_503;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 10s;
    }
}
```

If a backend returns 500 or times out, Nginx retries on the next server.

## Logging Which Backend Was Used

```nginx
log_format upstream '$remote_addr → $upstream_addr [$status] "$request"';
access_log /var/log/nginx/upstream.log upstream;
```

This logs `$upstream_addr` (the backend IP:port that handled the request) for every request.

## Checking Nginx Status

```nginx
server {
    listen 8080;
    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
curl http://localhost:8080/nginx_status
```

## Conclusion

Nginx round-robin is the default — just list servers in an `upstream` block without a `least_conn` or `ip_hash` directive. Add `max_fails` and `fail_timeout` for passive health detection. Use `proxy_next_upstream` to retry failed requests automatically. Log `$upstream_addr` to verify distribution in production.
