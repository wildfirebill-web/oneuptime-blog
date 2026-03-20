# How to Deploy Squid Proxy Cache via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Squid, Proxy, Docker, Caching

Description: Deploy Squid proxy server and caching layer using Portainer for forward proxy and HTTP caching capabilities.

## Introduction

Squid is a caching and forwarding HTTP proxy. It reduces bandwidth usage by caching frequently requested content, controls outbound access from internal networks, and can intercept and log HTTP traffic. This guide deploys Squid as a Portainer Stack.

## Prerequisites

- Portainer installed with Docker
- Understanding of forward proxy concepts

## Step 1: Create Squid Configuration

Before deploying, create the Squid configuration file on the host:

```bash
mkdir -p /opt/squid/conf
cat > /opt/squid/conf/squid.conf << 'EOF'
# /etc/squid/squid.conf

# ACLs

acl localnet src 10.0.0.0/8
acl localnet src 172.16.0.0/12
acl localnet src 192.168.0.0/16
acl SSL_ports port 443
acl CONNECT method CONNECT

# Allow local networks to use the proxy
http_access allow localnet
http_access allow localhost

# Block CONNECT on non-SSL ports
http_access deny CONNECT !SSL_ports

# Deny everything else
http_access deny all

# Proxy port
http_port 3128

# Cache settings
cache_mem 512 MB
maximum_object_size 50 MB
cache_dir ufs /var/spool/squid 4096 16 256

# Log format
access_log /var/log/squid/access.log squid
cache_log /var/log/squid/cache.log

# DNS servers
dns_nameservers 8.8.8.8 1.1.1.1
EOF
```

## Step 2: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Squid Proxy
version: "3.8"

services:
  squid:
    image: ubuntu/squid:latest
    container_name: squid
    restart: unless-stopped
    ports:
      - "3128:3128"    # HTTP proxy port
    volumes:
      - /opt/squid/conf/squid.conf:/etc/squid/squid.conf:ro
      - squid_cache:/var/spool/squid
      - squid_logs:/var/log/squid
    healthcheck:
      test: ["CMD", "squidclient", "-h", "localhost", "-p", "3128", "mgr:info"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - proxy_net

volumes:
  squid_cache:
  squid_logs:

networks:
  proxy_net:
    driver: bridge
```

## Step 3: Test the Proxy

```bash
# Test HTTP access through the proxy
curl -x http://localhost:3128 http://example.com

# Test HTTPS CONNECT tunnel
curl -x http://localhost:3128 https://example.com

# Check Squid access logs
docker exec squid tail -f /var/log/squid/access.log

# Check cache statistics
docker exec squid squidclient -h localhost -p 3128 mgr:info | grep -E "Requests|Hits|Misses"
```

## Step 4: Configure Applications to Use the Proxy

```bash
# Set proxy for curl
export http_proxy=http://squid-host:3128
export https_proxy=http://squid-host:3128
curl https://example.com

# Python requests
import requests
proxies = {'http': 'http://squid-host:3128', 'https': 'http://squid-host:3128'}
r = requests.get('https://api.example.com/data', proxies=proxies)

# Docker pull through proxy
docker pull --env HTTP_PROXY=http://squid-host:3128 alpine
```

## Step 5: Add Authentication

For controlled access, add basic authentication:

```bash
# Create password file on host
mkdir -p /opt/squid/passwords
docker run --rm ubuntu/squid htpasswd -bc /opt/squid/passwords/squid_users proxyuser securepassword
```

Add to `squid.conf`:

```text
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwords/squid_users
auth_param basic realm Squid Proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated
```

## Step 6: Monitor Cache Hit Rate

```bash
# View cache statistics
docker exec squid squidclient -h localhost -p 3128 mgr:counters | \
  grep -E "client_http|cache_hit"

# Or via cachemgr HTTP interface (add to squid.conf: http_access allow localhost)
curl http://localhost:3128/squid-internal-mgr/info
```

## Conclusion

Squid caches HTTP/HTTPS content, reducing bandwidth for environments with many clients accessing the same external resources (e.g., container image pulls, package downloads). The `cache_mem` parameter controls RAM used for hot objects, while `cache_dir ufs` stores larger objects on disk. Monitor the cache hit ratio - below 50% suggests most content is not cacheable, and you should review whether the cache is beneficial for your workload.
