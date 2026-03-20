# How to Configure DNS-Based IPv6 Load Balancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, Load Balancing, AAAA Records, High Availability

Description: A guide to implementing DNS-based load balancing for IPv6 services using multiple AAAA records, weighted responses, and health-check-based DNS.

## DNS Load Balancing for IPv6

DNS-based load balancing distributes traffic across multiple servers by returning multiple AAAA records for a single hostname. Clients pick one address and connect to it. This is simple to implement but has limitations compared to hardware or software load balancers.

## Basic Round-Robin AAAA Load Balancing

Add multiple AAAA records for the same hostname. DNS resolvers return all records; clients cycle through them:

```dns
; /var/named/example.com.zone
; Three IPv6 backends for www.example.com

www     300     IN  AAAA    2001:db8::1    ; Backend 1
www     300     IN  AAAA    2001:db8::2    ; Backend 2
www     300     IN  AAAA    2001:db8::3    ; Backend 3
```

Use short TTLs (300 seconds or less) for load-balanced records to enable quick failover.

## Verifying Round-Robin Behavior

```bash
# Run multiple queries to observe address rotation
for i in $(seq 1 5); do
    dig AAAA www.example.com @ns1.example.com +short
    echo "---"
done

# Observe that the order of addresses changes between queries
# BIND randomizes the answer order by default (rrset-order random)
```

## Configuring BIND for Round-Robin

BIND supports configurable `rrset-order` for controlling DNS record rotation:

```named
// /etc/named.conf
options {
    // Randomize the order of A/AAAA records in responses
    // Options: fixed, random, cyclic
    rrset-order {
        type AAAA;
        name "www.example.com";
        order random;  // random or cyclic for load balancing
    };
};
```

## Weighted DNS Load Balancing with PowerDNS Lua

PowerDNS supports Lua scripting for weighted responses. This enables proportional traffic distribution:

```lua
-- /etc/powerdns/pdns.lua
-- Weighted AAAA load balancing

-- Define backends with weights
-- Higher weight = more traffic
local backends = {
    {addr = "2001:db8::1", weight = 60},  -- 60% of traffic
    {addr = "2001:db8::2", weight = 30},  -- 30% of traffic
    {addr = "2001:db8::3", weight = 10},  -- 10% of traffic
}

-- Calculate total weight
local total = 0
for _, b in ipairs(backends) do total = total + b.weight end

function preresolve(dq)
    if dq.qname:equal("www.example.com") and dq.qtype == pdns.AAAA then
        -- Random weighted selection
        local rand = math.random(total)
        local cumulative = 0
        for _, b in ipairs(backends) do
            cumulative = cumulative + b.weight
            if rand <= cumulative then
                dq:addAnswer(pdns.AAAA, b.addr, 60)
                return true
            end
        end
    end
    return false
end
```

## Health-Check-Based DNS with nsupdate

For DNS load balancing that responds to server health, use a monitoring script that updates DNS when backends go down:

```bash
#!/bin/bash
# health-check-dns.sh - Update DNS when backends fail

ZONE="example.com"
HOSTNAME="www.example.com"
BACKENDS=("2001:db8::1" "2001:db8::2" "2001:db8::3")
TTL=60

update_dns() {
    local active_backends=("$@")

    # Build nsupdate commands
    local update_cmds="server 127.0.0.1\nzone $ZONE\n"
    update_cmds+="update delete $HOSTNAME. AAAA\n"

    for addr in "${active_backends[@]}"; do
        update_cmds+="update add $HOSTNAME. $TTL AAAA $addr\n"
    done
    update_cmds+="send\n"

    echo -e "$update_cmds" | nsupdate -k /etc/named/update.key
}

# Check each backend
active=()
for backend in "${BACKENDS[@]}"; do
    if ping6 -c 2 -W 2 "$backend" &>/dev/null; then
        active+=("$backend")
        echo "Backend $backend: UP"
    else
        echo "Backend $backend: DOWN - removing from DNS"
    fi
done

# Update DNS with only active backends
if [ ${#active[@]} -gt 0 ]; then
    update_dns "${active[@]}"
else
    echo "WARNING: All backends down! Keeping last known DNS state."
fi
```

Run via cron every minute for basic health-check DNS:

```bash
# /etc/crontab
* * * * * root /usr/local/bin/health-check-dns.sh
```

## Using anycast for IPv6 Load Balancing

For large-scale deployments, anycast routing provides true load balancing:

```bash
# Multiple servers announce the same IPv6 address via BGP
# Traffic is routed to the nearest announcing server

# Each server has the same anycast address configured
ip -6 addr add 2001:db8::service/128 dev lo

# And announces it via BGP (using BIRD as BGP daemon)
# /etc/bird/bird6.conf
protocol bgp upstream {
    local as 65001;
    neighbor 2001:db8::router as 65000;
    export filter {
        if net = 2001:db8::service/128 then accept;
        reject;
    };
}
```

## Summary

DNS-based IPv6 load balancing starts with multiple AAAA records (round-robin). For weighted distribution, use PowerDNS Lua scripting. For health-check-based DNS, use a monitoring script with `nsupdate` to add/remove AAAA records based on backend health. For production load balancing at scale, consider DNS as a coarse-grained method and complement it with anycast routing or a proper load balancer (HAProxy, nginx upstream) with full IPv6 support.
