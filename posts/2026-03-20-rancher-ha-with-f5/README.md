# How to Configure Rancher HA with F5

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, F5 BIG-IP, High Availability, Enterprise, Load Balancer, SSL

Description: Configure F5 BIG-IP as the enterprise load balancer for Rancher HA with iRules for health checking, persistence profiles, and SSL offloading configuration.

## Introduction

F5 BIG-IP is the enterprise standard for application delivery in regulated industries. Configuring it for Rancher HA requires creating virtual servers, pools with health monitors, and appropriate persistence profiles to ensure stable WebSocket connections for cluster agents.

## Prerequisites

- F5 BIG-IP LTM with appropriate license
- Management access to BIG-IP (TMSH or GUI)
- Three Rancher server nodes registered as pool members

## Step 1: Create Pool Members (Nodes)

```bash
# Using TMSH (Traffic Management Shell)

# Add Rancher server nodes to BIG-IP
tmsh create ltm node rancher-node-1 address 10.0.0.11
tmsh create ltm node rancher-node-2 address 10.0.0.12
tmsh create ltm node rancher-node-3 address 10.0.0.13
```

## Step 2: Create a Health Monitor

```bash
# Create an HTTPS health monitor for Rancher's /healthz endpoint
tmsh create ltm monitor https rancher-health-monitor {
    interval 10
    timeout 31
    recv "ok"
    send "GET /healthz HTTP/1.0\r\nHost: rancher.example.com\r\n\r\n"
    ssl-profile /Common/serverssl
}
```

## Step 3: Create the Rancher Pool

```bash
# Create pool with health monitor and load balancing algorithm
tmsh create ltm pool rancher-https-pool {
    load-balancing-mode least-connections-member
    monitor rancher-health-monitor
    members {
        rancher-node-1:443 { address 10.0.0.11 }
        rancher-node-2:443 { address 10.0.0.12 }
        rancher-node-3:443 { address 10.0.0.13 }
    }
}
```

## Step 4: Configure SSL Profile (SSL Passthrough)

For SSL passthrough (recommended for Rancher):

```bash
# Create a FastL4 profile for SSL passthrough
tmsh create ltm profile fastl4 rancher-fastl4 {
    idle-timeout 300
    reset-on-timeout disabled
}

# Create virtual server with FastL4 (bypasses SSL processing)
tmsh create ltm virtual rancher-https-vs {
    destination 10.0.0.10:443       # VIP address
    ip-protocol tcp
    pool rancher-https-pool
    profiles {
        rancher-fastl4 { }          # FastL4 enables passthrough
    }
    source-address-translation {
        type automap                # SNAT for return traffic
    }
}
```

## Step 5: Configure Kubernetes API Server Pool

```bash
# Pool for Kubernetes API (port 6443)
tmsh create ltm pool k8s-api-pool {
    load-balancing-mode least-connections-member
    monitor tcp
    members {
        rancher-node-1:6443 { address 10.0.0.11 }
        rancher-node-2:6443 { address 10.0.0.12 }
        rancher-node-3:6443 { address 10.0.0.13 }
    }
}

tmsh create ltm virtual k8s-api-vs {
    destination 10.0.0.10:6443
    ip-protocol tcp
    pool k8s-api-pool
    profiles {
        rancher-fastl4 { }
    }
    source-address-translation {
        type automap
    }
}
```

## Step 6: Configure Persistence for WebSocket Connections

Rancher uses WebSocket connections for cluster agents. Source-IP persistence ensures agents reconnect to the same Rancher pod:

```bash
# Create source address persistence profile
tmsh create ltm persistence source-addr rancher-persistence {
    timeout 3600    # 1 hour persistence
    match-across-pools enabled
}

# Apply to the pool
tmsh modify ltm virtual rancher-https-vs {
    persist { rancher-persistence { default yes } }
}
```

## Step 7: Save Configuration

```bash
# Save BIG-IP configuration
tmsh save /sys config
```

## Conclusion

F5 BIG-IP provides enterprise-grade load balancing for Rancher HA with advanced features like SSL bridging, application-layer health monitors, and granular persistence control. The source-address persistence profile is particularly important for stable WebSocket connections between Rancher agents and the server.
