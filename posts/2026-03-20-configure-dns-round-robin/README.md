# How to Configure DNS Round-Robin for Simple Load Balancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Round-Robin, Load Balancing, Linux, BIND, Configuration

Description: Configure DNS round-robin load balancing by creating multiple A records for the same hostname, with considerations for session persistence and health checking.

## Introduction

DNS round-robin distributes traffic across multiple servers by returning different IP addresses in rotating order for the same hostname. It requires no special load balancer hardware and works with any application. However, it has important limitations: no health checking, session stickiness is unreliable, and load distribution depends on client caching behavior. Understanding when DNS round-robin is appropriate guides its use.

## Configure Round-Robin in BIND

```bash
# Multiple A records for the same name = round-robin:
cat >> /etc/bind/zones/db.example.com << 'EOF'
; Round-robin across 3 web servers:
www     60  IN  A  10.20.0.10
www     60  IN  A  10.20.0.11
www     60  IN  A  10.20.0.12
; TTL=60: short TTL ensures clients re-query and get different IPs
; (avoids all clients sticking to same server due to long cache)
EOF

# Reload BIND:
rndc reload example.com

# Verify round-robin:
dig www.example.com +short
# Returns all 3 IPs; order may vary

dig www.example.com +short
# Second query may show different order (BIND shuffles by default)
```

## Configure rndc's Round-Robin Rotation

```bash
# BIND has a rrset-order directive to control answer ordering:
# /etc/bind/named.conf.options:
options {
    # Randomize order for each response (true round-robin):
    rrset-order { order random; };

    # Or: cyclic rotation (strict round-robin, not random):
    # rrset-order { order cyclic; };

    # Or: fixed order (not round-robin, always same):
    # rrset-order { order fixed; };
};
```

## Configure with dnsmasq

```bash
# dnsmasq automatically round-robins multiple address directives:
cat >> /etc/dnsmasq.d/lb.conf << 'EOF'
# Round-robin across 3 servers:
address=/api.example.com/10.20.0.10
address=/api.example.com/10.20.0.11
address=/api.example.com/10.20.0.12

# Short TTL for quicker distribution:
# dnsmasq uses the global TTL; set min-cache-ttl in conf
EOF
```

## TTL Considerations

```bash
# TTL determines how long clients cache a specific IP:
# TTL=3600: client keeps using same server for 1 hour
#           → Load is NOT distributed at client level, only per new connection
# TTL=60:   client re-queries every minute → better distribution
# TTL=0:    client must query on every request → maximum distribution
#           → Very high DNS query load, not recommended

# Best practice: TTL 30-60 for round-robin load balancing
# This balances re-query frequency against DNS server load

# Check effective TTL:
dig www.example.com | grep -A1 "ANSWER" | tail -1 | awk '{print "TTL:", $2}'
```

## Limitations of DNS Round-Robin

```bash
# 1. No health checking
# If a server goes down, DNS still returns its IP
# Clients get ECONNREFUSED until TTL expires and they re-query

# 2. Unequal distribution
# Clients cache differently; some may cache longer than TTL
# Some clients (DNS forwarders) serve many clients, amplifying one IP

# 3. Session persistence unreliable
# Subsequent queries may return different IPs
# Multi-request protocols (HTTP keep-alive is fine; HTTP/1.0 per-request may break)

# 4. Thundering herd
# When TTL expires simultaneously for many clients, all re-query
# And all get the same first IP in the rotation

# When to use DNS round-robin:
# - Stateless services (each request independent)
# - Batch jobs where session persistence doesn't matter
# - Geographic load distribution (A vs AAAA for different DCs)
# - Simple redundancy (acceptable that unhealthy server gets traffic briefly)
```

## Health-Checked Alternative

```bash
# For health checking: use nsupdate to remove unhealthy servers
# Simple health check + nsupdate script:

#!/bin/bash
SERVER_IP="10.20.0.12"
DOMAIN="www.example.com"
DNS_SERVER="10.20.0.1"

if ! curl -sf http://$SERVER_IP/health > /dev/null 2>&1; then
    echo "Server $SERVER_IP unhealthy, removing from DNS"
    nsupdate << EOF
server $DNS_SERVER
zone example.com
update delete $DOMAIN A $SERVER_IP
send
EOF
fi
```

## Conclusion

DNS round-robin is the simplest load balancing mechanism requiring no infrastructure beyond your existing DNS. Configure multiple A records with a short TTL (30-60 seconds) for reasonable distribution. Use BIND's `rrset-order random` for true randomization. Understand the limitations: no health checking, no true session persistence, and uneven distribution due to client-side caching. For production load balancing with health checks, DNS round-robin is a reasonable first step before investing in a proper load balancer, or as a complement to load balancers for geographic distribution.
