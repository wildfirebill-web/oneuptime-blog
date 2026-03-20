# How to Set Up HAProxy with Roundrobin Load Balancing for IPv4 Backends

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: HAProxy, Load Balancing, IPv4, Round Robin, Backend, Configuration

Description: Configure HAProxy to distribute IPv4 traffic evenly across multiple backend servers using the roundrobin algorithm, with weight-based load distribution.

## Introduction

Round robin is HAProxy's default load balancing algorithm. It sends each new connection to the next server in the list, cycling through all servers equally. Weighted round robin allows you to send more traffic to higher-capacity servers.

## Basic Roundrobin Configuration

```text
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    maxconn 50000

defaults
    mode http
    log global
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend web-frontend
    bind 0.0.0.0:80
    default_backend web-servers

backend web-servers
    balance roundrobin
    server web1 10.0.1.10:8080 check
    server web2 10.0.1.11:8080 check
    server web3 10.0.1.12:8080 check
```

Restart HAProxy:

```bash
sudo systemctl restart haproxy
```

## Weighted Round Robin

Assign different weights to balance unequal server capacities:

```text
backend web-servers
    balance roundrobin
    server web1 10.0.1.10:8080 check weight 10   # 50% of traffic
    server web2 10.0.1.11:8080 check weight 6    # 30% of traffic
    server web3 10.0.1.12:8080 check weight 4    # 20% of traffic
```

Weights are relative - total weight is 20 in this example. Server web1 receives 10/20 = 50%.

## Slow Start for Newly Enabled Servers

Prevent newly added servers from being immediately flooded:

```text
backend web-servers
    balance roundrobin
    server web1 10.0.1.10:8080 check slowstart 60s
    server web2 10.0.1.11:8080 check slowstart 60s
```

During the slowstart period (60 seconds), weight increases gradually from 0 to the configured weight.

## Verifying Round Robin Distribution

```bash
# Send 9 requests and check which server responds

for i in $(seq 1 9); do
  curl -s http://localhost/server-id
done

# Expected output (for 3 servers):
# web1, web2, web3, web1, web2, web3, web1, web2, web3
```

## HAProxy Stats to Monitor Distribution

```text
frontend stats
    bind 0.0.0.0:8404
    stats enable
    stats uri /stats
    stats refresh 5s
    stats show-legends
```

The stats page shows each server's session count and request rate, confirming equal distribution.

## Draining a Server Without Downtime

```bash
# Disable a server gracefully (wait for existing connections to finish)
echo "disable server web-servers/web2" | sudo socat stdio /run/haproxy/admin.sock

# Re-enable it
echo "enable server web-servers/web2" | sudo socat stdio /run/haproxy/admin.sock
```

## Leastconn vs Roundrobin

| Algorithm | Best For |
|---|---|
| roundrobin | Short, equal-duration requests |
| leastconn | Long-lived connections (DB, WebSocket) |
| source | Session persistence by client IP |
| random | Randomized even distribution |

## Conclusion

Configure HAProxy round robin by setting `balance roundrobin` in the backend section. Use `weight` for proportional distribution across unequal servers. Add `slowstart` to ramp traffic gradually to new servers. For long-lived connections, consider `leastconn` instead to avoid overloading servers that happen to receive connections first.
