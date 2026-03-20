# How to Configure keepalived with IPv6 VRRP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: keepalived, IPv6, VRRP, High Availability, Failover, Load Balancing

Description: A guide to configuring keepalived with VRRPv3 for IPv6 virtual IP addresses, providing high availability and automatic failover for IPv6 services.

keepalived implements VRRP (Virtual Router Redundancy Protocol) for high availability. VRRPv3 extends VRRP to support IPv6 virtual addresses. This guide covers configuring keepalived for IPv6 VIP failover with LVS/IPVS integration.

## VRRPv3 for IPv6

VRRPv3 is required for IPv6 (VRRPv2 only supports IPv4). keepalived supports VRRPv3 natively.

## Basic IPv6 VRRP Configuration

```nginx
# /etc/keepalived/keepalived.conf (MASTER node)

global_defs {
    router_id KEEPALIVED_MASTER
    # Enable VRRPv3 for IPv6
    vrrp_version 3
}

vrrp_instance VI_IPV6 {
    state MASTER
    interface eth0

    # VRRP ID (1-255, must match between MASTER and BACKUP)
    virtual_router_id 51

    # Priority (MASTER has highest)
    priority 150

    # Advertisement interval (seconds)
    advert_int 1

    # Authentication
    authentication {
        auth_type PASS
        auth_pass ipv6vrrp
    }

    # IPv6 Virtual IP address
    virtual_ipaddress {
        2001:db8::vip/64 dev eth0
        # Can add multiple VIPs:
        # 2001:db8::vip2/64 dev eth0
    }
}
```

```nginx
# /etc/keepalived/keepalived.conf (BACKUP node)

global_defs {
    router_id KEEPALIVED_BACKUP
    vrrp_version 3
}

vrrp_instance VI_IPV6 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100    # Lower priority than MASTER
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass ipv6vrrp
    }

    virtual_ipaddress {
        2001:db8::vip/64 dev eth0
    }
}
```

## Dual-Stack VRRP (IPv4 and IPv6)

```nginx
# VRRPv3 can carry both IPv4 and IPv6 virtual IPs

vrrp_instance VI_DUAL_STACK {
    state MASTER
    interface eth0
    virtual_router_id 52
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass dualpass
    }

    # Both IPv4 and IPv6 VIPs in same VRRP instance
    virtual_ipaddress {
        192.168.1.100/24 dev eth0
        2001:db8::100/64 dev eth0
    }
}
```

## VRRP with LVS Integration

```nginx
# VRRP + LVS for IPv6 load balancing with failover

vrrp_instance VI_IPV6_LB {
    state MASTER
    interface eth0
    virtual_router_id 53
    priority 150

    virtual_ipaddress {
        2001:db8::vip/64 dev eth0
    }
}

# IPv6 virtual server definition
virtual_server_group ipv6_group {
    2001:db8::vip 80
}

virtual_server group ipv6_group {
    delay_loop 5
    lb_algo rr
    lb_kind NAT
    protocol TCP

    # IPv6 real server
    real_server 2001:db8::server1 80 {
        weight 1
        HTTP_GET {
            url {
                path /health
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 2001:db8::server2 80 {
        weight 1
        HTTP_GET {
            url {
                path /health
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

## Failover Scripts

```nginx
# Run scripts on VRRP state transitions
vrrp_script check_service {
    script "/usr/local/bin/check_ipv6_service.sh"
    interval 5
    weight -20    # Decrease priority by 20 if script fails
    fall 2        # 2 failures to mark as failed
    rise 2        # 2 successes to mark as up
}

vrrp_instance VI_IPV6 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150

    virtual_ipaddress {
        2001:db8::vip/64 dev eth0
    }

    track_script {
        check_service
    }

    notify_master /usr/local/bin/master.sh
    notify_backup /usr/local/bin/backup.sh
    notify_fault  /usr/local/bin/fault.sh
}
```

## Managing keepalived

```bash
# Start keepalived
sudo systemctl start keepalived
sudo systemctl enable keepalived

# Check status
sudo systemctl status keepalived

# Verify VIP is on the correct node
ip -6 addr show eth0 | grep "2001:db8::vip"

# Tail logs
sudo journalctl -u keepalived -f

# Test failover: stop keepalived on MASTER
sudo systemctl stop keepalived
# VIP should move to BACKUP within 3 seconds
```

keepalived's VRRPv3 support makes it the standard tool for providing IPv6 virtual IP high availability on Linux, combining seamlessly with IPVS for complete IPv6 load balancing with automatic failover.
