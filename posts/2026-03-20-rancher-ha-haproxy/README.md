# How to Configure Rancher HA with HAProxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, has, HAProxy, High Availability, Load Balancing

Description: Configure HAProxy as the load balancer for Rancher HA deployments with health checks, SSL passthrough, and WebSocket support for reliable cluster management.

## Introduction

HAProxy is a high-performance, battle-tested load balancer commonly used in front of Rancher HA deployments. Its fine-grained configuration options make it ideal for handling Rancher's mix of HTTP/HTTPS traffic and long-lived WebSocket connections from cluster agents. This guide covers complete HAProxy configuration for Rancher HA.

## Prerequisites

- HAProxy 2.6+ installed on a dedicated node or VM
- Running Rancher HA cluster (3 nodes)
- TLS certificate for Rancher hostname
- HAProxy node with connectivity to all Rancher nodes

## Step 1: Install HAProxy

```bash
# Ubuntu/Debian

apt update
apt install -y haproxy

# RHEL/CentOS
yum install -y haproxy

# Verify version (2.6+ recommended)
haproxy -v
```

## Step 2: Configure HAProxy for Rancher

```haproxy
# /etc/haproxy/haproxy.cfg - Complete HAProxy configuration for Rancher

global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # TLS settings
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3
    tune.ssl.default-dh-param 2048

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 10s
    timeout client 3600s    # Long timeout for WebSocket connections
    timeout server 3600s
    timeout tunnel 3600s    # Critical for WebSocket/tunnel connections

#-------------------
# Rancher HTTPS
#-------------------
frontend rancher_https
    bind *:443 ssl crt /etc/haproxy/certs/rancher.pem
    mode http
    option forwardfor
    http-request set-header X-Forwarded-Proto https

    # WebSocket detection
    acl is_websocket hdr(Upgrade) -i websocket
    use_backend rancher_ws_backend if is_websocket
    default_backend rancher_https_backend

frontend rancher_http
    bind *:80
    mode http
    redirect scheme https code 301

#-------------------
# RKE2 API Server
#-------------------
frontend rke2_api
    bind *:6443
    mode tcp
    option tcplog
    default_backend rke2_api_backend

frontend rke2_register
    bind *:9345
    mode tcp
    option tcplog
    default_backend rke2_register_backend

#-------------------
# Backends
#-------------------
backend rancher_https_backend
    mode http
    option forwardfor
    option httpchk GET /ping
    http-check expect string pong
    balance roundrobin
    default-server inter 10s fall 3 rise 2

    server rancher-01 10.0.0.11:443 check ssl verify none
    server rancher-02 10.0.0.12:443 check ssl verify none
    server rancher-03 10.0.0.13:443 check ssl verify none

backend rancher_ws_backend
    mode http
    option http-server-close
    option forwardfor
    balance source  # Sticky sessions for WebSocket
    timeout tunnel 3600s

    server rancher-01 10.0.0.11:443 check ssl verify none
    server rancher-02 10.0.0.12:443 check ssl verify none
    server rancher-03 10.0.0.13:443 check ssl verify none

backend rke2_api_backend
    mode tcp
    balance roundrobin
    option ssl-hello-chk

    server cp-01 10.0.0.11:6443 check
    server cp-02 10.0.0.12:6443 check
    server cp-03 10.0.0.13:6443 check

backend rke2_register_backend
    mode tcp
    balance roundrobin

    server cp-01 10.0.0.11:9345 check
    server cp-02 10.0.0.12:9345 check
    server cp-03 10.0.0.13:9345 check

#-------------------
# HAProxy Stats
#-------------------
frontend stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:securepassword
    stats hide-version
```

## Step 3: Configure SSL Certificate

```bash
# Combine certificate and private key for HAProxy
cat /etc/ssl/certs/rancher.crt \
    /etc/ssl/private/rancher.key \
    > /etc/haproxy/certs/rancher.pem

chmod 600 /etc/haproxy/certs/rancher.pem
```

## Step 4: Configure SSL Passthrough (Alternative)

```haproxy
# For SSL passthrough (Rancher handles TLS termination)
frontend rancher_ssl_passthrough
    bind *:443
    mode tcp
    option tcplog
    use_backend rancher_passthrough

backend rancher_passthrough
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    # Health check on TCP level
    option tcp-check
    tcp-check connect ssl

    server rancher-01 10.0.0.11:443 check
    server rancher-02 10.0.0.12:443 check
    server rancher-03 10.0.0.13:443 check
```

## Step 5: Enable and Verify HAProxy

```bash
# Validate configuration
haproxy -c -f /etc/haproxy/haproxy.cfg

# Start HAProxy
systemctl enable haproxy
systemctl start haproxy

# Check HAProxy status
systemctl status haproxy

# View stats
curl http://localhost:8404/stats

# Test Rancher health through HAProxy
curl -sk https://rancher.example.com/ping
```

## Step 6: Configure HAProxy High Availability with Keepalived

```bash
# Run HAProxy on 2+ nodes with keepalived VIP
# haproxy-primary keepalived.conf
vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101

    track_script {
        chk_haproxy
    }

    virtual_ipaddress {
        10.0.0.100/24
    }
}
```

## Step 7: Monitor HAProxy

```bash
# HAProxy stats via socket
echo "show stat" | socat unix-connect:/run/haproxy/admin.sock stdio | \
  cut -d',' -f1,2,18,19,21 | grep rancher

# Show current connections
echo "show info" | socat unix-connect:/run/haproxy/admin.sock stdio | \
  grep -E "CurrConns|MaxConn|Uptime"

# Disable a server for maintenance
echo "disable server rancher_https_backend/rancher-01" | \
  socat unix-connect:/run/haproxy/admin.sock stdio
```

## Conclusion

HAProxy provides enterprise-grade load balancing for Rancher HA with extensive health checking, WebSocket support, and real-time monitoring. The critical configuration elements are generous timeout settings for WebSocket connections (3600s), SSL certificate handling, and proper health check configuration using Rancher's `/ping` endpoint. For production deployments, run HAProxy in an active-passive pair with keepalived to eliminate the load balancer itself as a single point of failure.
