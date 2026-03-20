# How to Configure Rancher HA with External Load Balancer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, High Availability, Load Balancer, Kubernetes, TLS Termination, Networks

Description: Configure an external load balancer for Rancher HA to distribute traffic across multiple server nodes with health checks and TLS passthrough.

## Introduction

An external load balancer in front of Rancher HA serves two purposes: distributing incoming HTTPS traffic across Rancher Server nodes, and providing a single stable endpoint for downstream cluster agents and users. This guide covers load balancer configuration for Rancher HA deployments.

## TLS Termination Strategy

Rancher supports two load balancer TLS modes:

1. **TLS Passthrough**: Load balancer forwards raw TCP; Rancher handles TLS. Simpler but requires Layer 4 (TCP) load balancing.
2. **TLS Termination**: Load balancer handles TLS; forwards HTTP to Rancher. Requires Rancher to be configured with `--set tls=external`.

TLS Passthrough is recommended for production as it preserves end-to-end encryption.

## Option 1: Cloud Load Balancer (AWS NLB)

```bash
# Create an AWS Network Load Balancer with TCP listeners

aws elbv2 create-load-balancer \
  --name rancher-nlb \
  --type network \
  --scheme internet-facing \
  --subnets subnet-abc123 subnet-def456 subnet-ghi789

# Create target group for HTTPS/443
aws elbv2 create-target-group \
  --name rancher-servers \
  --protocol TCP \
  --port 443 \
  --vpc-id vpc-12345678 \
  --health-check-protocol HTTPS \
  --health-check-path /healthz \
  --health-check-interval-seconds 10 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2

# Register targets (Rancher server IPs)
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=10.0.0.11 Id=10.0.0.12 Id=10.0.0.13
```

## Option 2: HAProxy Configuration

```haproxy
# /etc/haproxy/haproxy.cfg

global
    maxconn 50000
    log stdout local0

defaults
    log global
    option tcplog
    timeout connect 5s
    timeout client 50s
    timeout server 50s

# Rancher HTTPS frontend (TLS passthrough)
frontend rancher-https
    bind *:443
    mode tcp
    default_backend rancher-servers

backend rancher-servers
    mode tcp
    balance roundrobin
    option tcp-check
    # Health check via HTTPS
    server rancher-1 10.0.0.11:443 check check-ssl verify none
    server rancher-2 10.0.0.12:443 check check-ssl verify none
    server rancher-3 10.0.0.13:443 check check-ssl verify none

# K3s/RKE2 API server frontend
frontend k8s-api
    bind *:6443
    mode tcp
    default_backend k8s-api-servers

backend k8s-api-servers
    mode tcp
    balance roundrobin
    server server-1 10.0.0.11:6443 check
    server server-2 10.0.0.12:6443 check
    server server-3 10.0.0.13:6443 check
```

## Option 3: Keepalived for VIP

For on-premises HA, Keepalived provides a virtual IP that floats between nodes:

```bash
# /etc/keepalived/keepalived.conf on each server

vrrp_script check_rancher {
    script "curl -k -s https://localhost/healthz"
    interval 3
    weight 10
}

vrrp_instance VI_RANCHER {
    state MASTER        # Set to BACKUP on other nodes
    interface eth0
    virtual_router_id 51
    priority 100        # Highest priority = master; lower on other nodes
    advert_int 1

    track_script {
        check_rancher
    }

    virtual_ipaddress {
        10.0.0.10/24    # Virtual IP shared between all nodes
    }
}
```

## Health Check Configuration

```bash
# Rancher health endpoint (always returns 200 if healthy)
curl -k https://rancher.example.com/healthz
# Response: ok

# Detailed health with version info
curl -k https://rancher.example.com/v3/settings/server-version
```

## Conclusion

The external load balancer is the entry point for all Rancher management traffic. TLS passthrough with TCP health checks is the recommended approach for production. The Kubernetes API port (6443 for RKE2, 6443 for K3s) should also be load-balanced to allow kubectl and Rancher agents to reach any healthy server node.
