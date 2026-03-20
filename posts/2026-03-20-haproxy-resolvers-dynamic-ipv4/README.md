# How to Use HAProxy resolvers Section for Dynamic IPv4 Server Discovery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: HAProxy, DNS, Resolver, IPv4, Service Discovery, Dynamic Configuration

Description: Configure the HAProxy resolvers section to enable dynamic DNS-based server discovery, automatically updating IPv4 backend addresses when DNS records change.

## Introduction

The `resolvers` section in HAProxy enables runtime DNS resolution for backend servers. As services scale or fail over, HAProxy picks up new IPv4 addresses from DNS without requiring a configuration reload-ideal for containerized and cloud-native environments.

## Basic Resolvers Configuration

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    maxconn 50000

resolvers local_dns
    # DNS server addresses
    nameserver dns1 127.0.0.53:53     # systemd-resolved (Ubuntu)
    nameserver dns2 10.0.0.53:53      # Internal DNS server

    # Resolution settings
    resolve_retries 3                  # Retries before considering DNS down
    timeout resolve 1s                 # Timeout per DNS query
    timeout retry   1s                 # Timeout between retries

    # How long to hold different DNS response states
    hold valid    10s    # Valid responses cached for 10s (even if TTL is shorter)
    hold other    30s    # Non-NXDOMAIN errors held for 30s
    hold refused  30s
    hold nx       30s
    hold timeout  10s
    hold obsolete 30s

    # Accept IP changes without requiring full reload
    accepted_payload_size 8192
```

## Backend Using Dynamic DNS Resolution

```haproxy
backend microservices
    balance roundrobin

    # The 'init-addr' parameter controls initial address resolution:
    # - last: use last known address
    # - libc: use OS resolver
    # - none: fail if DNS is unavailable at startup
    server svc1 user-service.default.svc.cluster.local:8080 \
        check resolvers local_dns resolve-prefer ipv4 init-addr last

    server svc2 order-service.default.svc.cluster.local:8080 \
        check resolvers local_dns resolve-prefer ipv4 init-addr last
```

## Server Templates for Scale-Out Discovery

Use `server-template` to automatically create multiple servers from DNS results:

```haproxy
backend scalable_backend
    balance roundrobin

    # Create up to 10 server entries from DNS A records
    # HAProxy populates server1 through server10 from DNS responses
    server-template svc 1-10 myapp.service.consul:8080 \
        check resolvers local_dns resolve-prefer ipv4 init-addr none
```

## Consul Service Discovery Integration

For Consul-based service discovery with DNS:

```haproxy
resolvers consul_dns
    nameserver consul1 127.0.0.1:8600   # Consul DNS port
    hold valid    5s
    hold other    5s
    timeout resolve 2s

backend consul_services
    server-template web 1-5 web.service.consul:80 \
        check resolvers consul_dns resolve-prefer ipv4 init-addr none
```

## Monitoring DNS Resolution Activity

```bash
# View resolver status and cached entries

echo "show resolvers" | sudo socat stdio /run/haproxy/admin.sock

# View server states and their current resolved IPs
echo "show servers state" | sudo socat stdio /run/haproxy/admin.sock

# Force re-resolution of a specific server
echo "set server microservices/svc1 fqdn user-service.default.svc.cluster.local" | \
  sudo socat stdio /run/haproxy/admin.sock

# Force DNS query immediately (bypass cache)
echo "set server microservices/svc1 fqdn user-service.default.svc.cluster.local" | \
  sudo socat stdio /run/haproxy/admin.sock
```

## HAProxy Logs for DNS Events

```bash
# Monitor HAProxy logs for DNS resolution events
sudo tail -f /var/log/haproxy.log | grep -i "resolv\|DNS\|FQDN"

# Expected log entries:
# Server microservices/svc1 changed its IP from 10.0.0.10 to 10.0.0.15
# health check for server microservices/svc1 succeeded
```

## Conclusion

The HAProxy `resolvers` section transforms static IP backends into dynamically discovered services. Configure sensible `hold` timeouts to avoid excessive DNS queries while still reacting quickly to infrastructure changes. Use `server-template` for auto-scaling backends and `init-addr none` for container environments where services may not exist at HAProxy startup. This approach removes the need for config reloads when backend IPs change.
