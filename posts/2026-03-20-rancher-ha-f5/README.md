# How to Configure Rancher HA with F5

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, HA, F5, BIG-IP, High Availability, Enterprise

Description: Configure F5 BIG-IP as the enterprise load balancer for Rancher HA deployments with iRules, health monitors, and SSL offloading for mission-critical environments.

## Introduction

F5 BIG-IP is a common choice in enterprise environments for load balancing Rancher HA deployments. Its advanced traffic management capabilities, comprehensive health monitoring, and iRule scripting provide the fine-grained control needed for complex enterprise requirements. This guide covers F5 configuration for Rancher HA.

## Prerequisites

- F5 BIG-IP LTM (version 14.x or later)
- Access to F5 management interface (TMSH or GUI)
- Running Rancher HA cluster (3 nodes)
- TLS certificate imported into F5
- F5 admin credentials

## Step 1: Create Health Monitor

```tcl
# TMSH commands for F5 configuration

# Create HTTPS health monitor for Rancher

tmsh create ltm monitor https rancher_https_monitor {
    defaults-from https
    interval 10
    timeout 31
    send "GET /ping HTTP/1.1\r\nHost: rancher.example.com\r\nConnection: Close\r\n\r\n"
    recv "pong"
    recv-disable ""
    ssl-profile /Common/serverssl
}
```

## Step 2: Create Node Objects

```tcl
# Create node objects for Rancher servers
tmsh create ltm node rancher-01 address 10.0.0.11
tmsh create ltm node rancher-02 address 10.0.0.12
tmsh create ltm node rancher-03 address 10.0.0.13

# Create node for RKE2 API servers (same nodes, different ports)
# (Nodes are shared across pools)
```

## Step 3: Create Pool for Rancher HTTPS

```tcl
# Create pool for Rancher web traffic
tmsh create ltm pool rancher_https_pool {
    members {
        rancher-01:443 { address 10.0.0.11 }
        rancher-02:443 { address 10.0.0.12 }
        rancher-03:443 { address 10.0.0.13 }
    }
    load-balancing-mode least-connections-member
    monitor rancher_https_monitor
    service-down-action none
}

# Create pool for RKE2 API server
tmsh create ltm pool rke2_api_pool {
    members {
        rancher-01:6443 { address 10.0.0.11 }
        rancher-02:6443 { address 10.0.0.12 }
        rancher-03:6443 { address 10.0.0.13 }
    }
    load-balancing-mode least-connections-member
    monitor tcp_half_open
}

# Create pool for RKE2 registration
tmsh create ltm pool rke2_register_pool {
    members {
        rancher-01:9345 { address 10.0.0.11 }
        rancher-02:9345 { address 10.0.0.12 }
        rancher-03:9345 { address 10.0.0.13 }
    }
    load-balancing-mode least-connections-member
    monitor tcp
}
```

## Step 4: Configure SSL Profile

```tcl
# Import certificate and key to F5
# (Use GUI to import, or use TMSH with base64-encoded cert)

# Create server SSL profile for backend connections to Rancher
tmsh create ltm profile server-ssl rancher_server_ssl {
    defaults-from serverssl
    # Disable certificate verification if using self-signed
    authenticate once
    peer-cert-mode ignore
}

# Create client SSL profile for frontend (client to F5)
tmsh create ltm profile client-ssl rancher_client_ssl {
    defaults-from clientssl
    cert-key-chain {
        rancher {
            cert rancher.crt
            key rancher.key
            chain rancher-chain.crt
        }
    }
    # Modern TLS only
    options { no-sslv3 no-tlsv1 no-tlsv1.1 }
    ciphers "ECDHE+AES128+AESGCM:ECDHE+AES256+AESGCM"
}
```

## Step 5: Create iRule for WebSocket Handling

```tcl
# iRule: handle WebSocket upgrades and long connections
when HTTP_REQUEST {
    # Detect WebSocket upgrade
    if { [HTTP::header "Upgrade"] eq "websocket" } {
        # Use source affinity for WebSocket connections
        persist source_addr 3600
    }

    # Set extended timeout for long-lived connections
    TCP::idletime 3600
}

when HTTP_RESPONSE {
    # Pass through WebSocket upgrade responses
    if { [HTTP::header "Upgrade"] eq "websocket" } {
        # WebSocket connection established
        log local0. "WebSocket connection: [IP::remote_addr]"
    }
}

# Create the iRule
tmsh create ltm rule rancher_websocket_irule {
    when HTTP_REQUEST {
        if { [HTTP::header "Upgrade"] eq "websocket" } {
            persist source_addr 3600
        }
        TCP::idletime 3600
    }
}
```

## Step 6: Create Virtual Servers

```tcl
# Rancher HTTPS virtual server (with SSL offloading)
tmsh create ltm virtual rancher_https_vs {
    destination 10.0.0.100:443
    ip-protocol tcp
    pool rancher_https_pool
    profiles {
        rancher_client_ssl { context clientside }
        rancher_server_ssl { context serverside }
        http {}
        websocket {}
        tcp {}
    }
    rules {
        rancher_websocket_irule
    }
    source-address-translation {
        type automap
    }
    translate-address enabled
    translate-port enabled
}

# HTTP to HTTPS redirect
tmsh create ltm virtual rancher_http_vs {
    destination 10.0.0.100:80
    ip-protocol tcp
    rules {
        _sys_https_redirect
    }
    profiles {
        http {}
        tcp {}
    }
}

# RKE2 API server (TCP passthrough)
tmsh create ltm virtual rke2_api_vs {
    destination 10.0.0.100:6443
    ip-protocol tcp
    pool rke2_api_pool
    profiles {
        tcp {}
    }
    translate-address enabled
}

# RKE2 registration
tmsh create ltm virtual rke2_register_vs {
    destination 10.0.0.100:9345
    ip-protocol tcp
    pool rke2_register_pool
    profiles {
        tcp {}
    }
    translate-address enabled
}
```

## Step 7: Enable Connection Persistence

```tcl
# Create source address persistence profile for WebSocket
tmsh create ltm persistence source-addr rancher_src_persistence {
    defaults-from source_addr
    timeout 3600
    match-across-services enabled
}

# Apply to the virtual server
tmsh modify ltm virtual rancher_https_vs {
    persist {
        rancher_src_persistence { default yes }
    }
}
```

## Step 8: Save and Verify Configuration

```bash
# Save F5 configuration
tmsh save sys config

# Check pool member status
tmsh show ltm pool rancher_https_pool

# Check virtual server statistics
tmsh show ltm virtual rancher_https_vs

# Test from F5 shell
curl -sk https://10.0.0.100/ping
```

## Conclusion

F5 BIG-IP provides enterprise-grade capabilities for Rancher HA, including advanced health monitoring, iRule scripting for WebSocket handling, and SSL offloading with hardware acceleration. The WebSocket iRule ensures that long-lived cluster agent connections remain stable, and source persistence prevents WebSocket connection interruptions during load balancing decisions. For enterprises already invested in F5 infrastructure, this approach leverages existing tooling and expertise while delivering the reliability expected in mission-critical environments.
