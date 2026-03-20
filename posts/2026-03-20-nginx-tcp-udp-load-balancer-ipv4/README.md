# How to Set Up Nginx as a TCP/UDP Load Balancer for IPv4 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, TCP, UDP, Load Balancing, IPv4, Stream Module

Description: Configure Nginx as a Layer 4 TCP and UDP load balancer using the stream module to distribute IPv4 traffic for databases, DNS, and other non-HTTP protocols.

## Introduction

Nginx's `stream` module enables Layer 4 (TCP/UDP) load balancing. This is separate from the `http` module and handles any TCP or UDP protocol — MySQL, PostgreSQL, Redis, DNS, MQTT, and more. The stream block goes in the main `nginx.conf`.

## Enabling the Stream Module

The stream module is included in most Nginx installations. Verify:

```bash
nginx -V 2>&1 | grep stream
# Should show: --with-stream
```

## TCP Load Balancing

```nginx
# /etc/nginx/nginx.conf

stream {
    upstream mysql-backends {
        server 10.0.2.10:3306;
        server 10.0.2.11:3306;
        server 10.0.2.12:3306;
    }

    server {
        listen 3306;
        proxy_pass mysql-backends;
        proxy_connect_timeout 5s;
        proxy_timeout 300s;
    }

    upstream redis-backends {
        server 10.0.3.10:6379;
        server 10.0.3.11:6379;
    }

    server {
        listen 6379;
        proxy_pass redis-backends;
    }
}
```

## UDP Load Balancing (DNS Example)

```nginx
stream {
    upstream dns-servers {
        server 8.8.8.8:53;
        server 1.1.1.1:53;
    }

    server {
        listen 53 udp;
        proxy_pass dns-servers;
        proxy_timeout 1s;
        proxy_responses 1;   # UDP: number of responses to wait for
    }
}
```

## Including Stream Config from a Separate File

```nginx
# /etc/nginx/nginx.conf
http {
    include /etc/nginx/conf.d/*.conf;
}

stream {
    include /etc/nginx/stream.d/*.conf;
}
```

```nginx
# /etc/nginx/stream.d/postgres.conf

upstream pg-pool {
    least_conn;
    server 10.0.2.20:5432 weight=2;
    server 10.0.2.21:5432 weight=1;
}

server {
    listen 5432;
    proxy_pass pg-pool;
    proxy_connect_timeout 3s;
    proxy_timeout 600s;
}
```

## Health Checks (Nginx Plus)

Nginx OSS uses passive checks (mark server down after proxy errors):

```nginx
stream {
    upstream tcp-backends {
        server 10.0.1.10:8080 max_fails=2 fail_timeout=30s;
        server 10.0.1.11:8080 max_fails=2 fail_timeout=30s;
    }
}
```

## SSL Passthrough (TCP)

Pass SSL/TLS traffic directly to backends without termination:

```nginx
stream {
    upstream https-backends {
        server 10.0.1.10:443;
        server 10.0.1.11:443;
    }

    server {
        listen 443;
        proxy_pass https-backends;
        ssl_preread on;   # Read SNI to route to correct upstream
    }
}
```

## SSL Termination at the Stream Level

```nginx
stream {
    upstream db-backends {
        server 10.0.2.10:5432;
    }

    server {
        listen 5432 ssl;
        ssl_certificate     /etc/ssl/certs/server.crt;
        ssl_certificate_key /etc/ssl/private/server.key;
        proxy_pass db-backends;
    }
}
```

## Logging for Stream

```nginx
stream {
    log_format stream '$remote_addr → $upstream_addr [$time_local] '
                      '$protocol $status $bytes_sent $bytes_received '
                      '$session_time';

    access_log /var/log/nginx/stream.log stream;

    upstream mysql-backends {
        server 10.0.2.10:3306;
    }

    server {
        listen 3306;
        proxy_pass mysql-backends;
    }
}
```

## Conclusion

Configure the `stream` block in `nginx.conf` (not inside `http`) to enable Layer 4 load balancing. Use `listen <port>` for TCP and `listen <port> udp` for UDP. The `proxy_pass` directive routes to an upstream pool. Use `least_conn` for connection-oriented protocols like databases, and `ssl_preread on` for SNI-based routing of SSL traffic.
