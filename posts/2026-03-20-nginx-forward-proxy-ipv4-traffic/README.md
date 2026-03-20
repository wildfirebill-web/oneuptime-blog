# How to Set Up Nginx as a Forward Proxy for IPv4 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, Forward Proxy, IPv4, HTTP, HTTPS, Ngx_http_proxy_module, Networking

Description: Configure Nginx as an HTTP forward proxy for IPv4 clients, enabling web traffic routing, access control, and content filtering through a centralized proxy.

## Introduction

A forward proxy handles outbound requests from clients to external servers. While Nginx is primarily used as a reverse proxy, it can act as a forward proxy for HTTP traffic using the `proxy_pass` directive combined with the `$http_host` variable. For HTTPS CONNECT tunneling, you'll need the `ngx_http_proxy_connect_module`.

## HTTP Forward Proxy Configuration

```nginx
# /etc/nginx/nginx.conf

http {
    
    # Resolver for proxy DNS lookups
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    
    server {
        listen 3128;      # Standard proxy port
        
        # Only allow internal IPv4 clients
        allow 10.0.0.0/8;
        allow 172.16.0.0/12;
        allow 192.168.0.0/16;
        deny all;
        
        # Forward proxy for HTTP requests
        location / {
            # $http_host is the Host header from the client request
            # This enables dynamic proxy target based on requested hostname
            proxy_pass http://$http_host;
            
            # Forward the original host to the backend
            proxy_set_header Host $http_host;
            
            # Pass real client IP (for servers that log it)
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            # Connection settings
            proxy_connect_timeout 10s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            
            # Don't buffer large responses (stream them)
            proxy_buffering off;
        }
        
        # Log proxy traffic
        access_log /var/log/nginx/proxy_access.log;
        error_log  /var/log/nginx/proxy_error.log;
    }
}
```

## Blocking Specific Domains

```nginx
http {
    # Domain blocklist using map
    map $http_host $blocked {
        default                         0;
        "~(facebook|instagram|tiktok)" 1;   # Block social media
        "~\.example-blocked\.com$"      1;
    }
    
    server {
        listen 3128;
        allow 192.168.0.0/16;
        deny all;
        
        location / {
            # Return 403 for blocked domains
            if ($blocked) {
                return 403 "Access to $http_host is blocked by policy.\n";
            }
            
            proxy_pass http://$http_host;
            proxy_set_header Host $http_host;
            proxy_connect_timeout 10s;
            proxy_read_timeout 30s;
        }
    }
}
```

## Adding Basic Proxy Authentication

```nginx
http {
    server {
        listen 3128;
        
        # Require authentication
        auth_basic "Proxy Authentication Required";
        auth_basic_user_file /etc/nginx/proxy-users;
        
        location / {
            proxy_pass http://$http_host;
            proxy_set_header Host $http_host;
        }
    }
}
```

Create the password file:

```bash
sudo htpasswd -c /etc/nginx/proxy-users proxyuser
```

## Configuring Clients to Use the Proxy

```bash
# System-wide proxy settings

export http_proxy="http://proxy.example.com:3128"
export https_proxy="http://proxy.example.com:3128"
export no_proxy="localhost,127.0.0.1,*.internal.example.com"

# Test the proxy
curl -x http://proxy.example.com:3128 http://httpbin.org/ip

# With authentication
curl -x http://proxyuser:password@proxy.example.com:3128 http://example.com
```

## HTTPS Forward Proxy with ngx_http_proxy_connect_module

For HTTPS, clients send HTTP CONNECT to tunnel through the proxy. This requires a module that is not in the standard Nginx package:

```bash
# Install nginx with the proxy_connect module (varies by distribution)
# Or build from source with the module

# Test CONNECT method support
curl -v -x http://proxy.example.com:3128 https://example.com
```

## Logging Proxy Usage

```nginx
http {
    log_format proxy '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'upstream=$upstream_addr time=$upstream_response_time';
    
    server {
        listen 3128;
        access_log /var/log/nginx/proxy.log proxy;
        
        location / {
            proxy_pass http://$http_host;
        }
    }
}
```

## Applying the Configuration

```bash
# Validate
sudo nginx -t

# Reload
sudo nginx -s reload
```

## Conclusion

Nginx makes a capable HTTP forward proxy for internal networks with domain filtering and access control. For full HTTPS CONNECT tunneling in production, use a dedicated proxy solution like Squid or a patched Nginx build with the `ngx_http_proxy_connect_module`.
