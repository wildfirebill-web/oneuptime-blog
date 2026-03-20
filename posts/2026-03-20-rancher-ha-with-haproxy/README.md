# How to Configure Rancher HA with HAProxy - With

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, HAProxy, High Availability, Load Balancer, TCP, Networks

Description: Configure HAProxy as the front-end load balancer for Rancher HA with health checks, SSL passthrough, and statistics monitoring.

## Introduction

HAProxy is a high-performance, battle-tested TCP and HTTP load balancer. For on-premises Rancher HA deployments, HAProxy provides the traffic distribution and health checking needed to ensure continuous availability across multiple Rancher server nodes.

## Step 1: Install HAProxy

```bash
# Ubuntu/Debian

apt-get install -y haproxy

# RHEL/CentOS
yum install -y haproxy

# Verify version (2.4+ recommended)
haproxy -v
```

## Step 2: Configure HAProxy for Rancher

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log    local0
    log /dev/log    local1 notice
    maxconn 50000
    user haproxy
    group haproxy

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5s
    timeout client  1m
    timeout server  1m

# Statistics dashboard
listen stats
    bind *:8080
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:strongpassword

# Rancher HTTPS (SSL Passthrough)
frontend rancher_https_frontend
    bind *:443
    mode tcp
    option tcplog
    tcp-request inspect-delay 5s
    default_backend rancher_https_backend

backend rancher_https_backend
    mode tcp
    balance leastconn    # Route to server with fewest active connections
    option ssl-hello-chk  # Verify SSL is accepting connections

    server rancher-node-1 10.0.0.11:443 check weight 1 maxconn 1000
    server rancher-node-2 10.0.0.12:443 check weight 1 maxconn 1000
    server rancher-node-3 10.0.0.13:443 check weight 1 maxconn 1000

# Kubernetes API Server
frontend k8s_api_frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s_api_backend

backend k8s_api_backend
    mode tcp
    balance roundrobin
    option tcp-check

    server k8s-node-1 10.0.0.11:6443 check
    server k8s-node-2 10.0.0.12:6443 check
    server k8s-node-3 10.0.0.13:6443 check
```

## Step 3: Enable and Start HAProxy

```bash
# Validate configuration syntax
haproxy -c -f /etc/haproxy/haproxy.cfg

# Enable and start the service
systemctl enable haproxy
systemctl start haproxy

# Check status
systemctl status haproxy
```

## Step 4: Configure HAProxy High Availability

Run HAProxy on two hosts with Keepalived for HA:

```bash
# /etc/keepalived/keepalived.conf (on HAProxy host 1)
vrrp_script check_haproxy {
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
        check_haproxy
    }
    virtual_ipaddress {
        10.0.0.10    # Floating VIP
    }
}
```

## Step 5: Monitor HAProxy Stats

Access the HAProxy statistics page at `http://haproxy-host:8080/stats` to monitor:
- Session rates per backend
- Current active connections
- Backend server health (green = UP, red = DOWN)
- Response times

## Step 6: Test Failover

```bash
# Verify load balancing is working
curl -k https://10.0.0.10/healthz    # Should return "ok"

# Simulate node failure
ssh 10.0.0.11 "sudo systemctl stop rke2-server"

# Rancher should still respond through remaining nodes
curl -k https://10.0.0.10/healthz    # Should still return "ok"
```

## Conclusion

HAProxy is a reliable, lightweight choice for on-premises Rancher HA load balancing. The `ssl-hello-chk` health check option ensures HAProxy only routes to nodes where TLS is properly responding, providing more accurate health detection than a simple TCP connect check.
