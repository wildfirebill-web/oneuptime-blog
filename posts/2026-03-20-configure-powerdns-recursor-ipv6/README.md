# How to Configure PowerDNS Recursor with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PowerDNS, Recursor, DNS, IPv6, Recursive, Pdns-recursor, DNSSEC

Description: Configure PowerDNS Recursor to listen on IPv6, resolve queries using IPv6 transport, and apply access controls for IPv6 client networks.

## Introduction

PowerDNS Recursor (pdns-recursor) is a high-performance recursive resolver. It supports IPv6 natively for both incoming queries and outbound resolution, making it suitable for dual-stack and IPv6-only networks.

## Installation

```bash
apt-get install -y pdns-recursor
```

## Step 1: recursor.conf for IPv6

```ini
# /etc/powerdns/recursor.conf

# Listen on all IPv4 and IPv6 interfaces

local-address=0.0.0.0, ::

# Or specific addresses
# local-address=127.0.0.1, ::1, 2001:db8::53

local-port=53

# Allow queries from these networks
allow-from=127.0.0.0/8, ::1/128, 10.0.0.0/8, 192.168.0.0/16, 2001:db8::/32, fe80::/10

# Use IPv6 for outbound when available
# (default: yes if system has IPv6)
```

## Step 2: Performance

```ini
# Threads and caching
threads=4
max-cache-entries=2000000
max-packetcache-entries=500000

# Prefetch expiring records
serve-rfc1918=yes

# Logging
loglevel=5
log-common-errors=yes
```

## Step 3: Forward Zones

```ini
# Forward internal zone to IPv6 nameserver
forward-zones=example.internal=2001:db8:1::53

# Forward all queries to upstream
# (makes it a forwarding resolver rather than full recursive)
forward-zones-recurse=.=2606:4700:4700::1111;2606:4700:4700::1001;8.8.8.8
```

## Step 4: DNSSEC Validation

```ini
# Enable DNSSEC validation
dnssec=validate

# Download root trust anchor
# pdns-recursor automatically fetches it if not present
```

## Step 5: Lua Scripting for IPv6-Aware Filtering

```lua
-- /etc/powerdns/filter.lua
-- Block queries for known malicious domains from IPv6 clients

function preresolve(dq)
    -- Log the query source
    if dq.remoteaddr:isIPv6() then
        pdnslog("IPv6 query from " .. dq.remoteaddr:toString() ..
                " for " .. dq.qname:toString())
    end

    -- Block specific domain
    if dq.qname:equal("malware.example.com") then
        dq.rcode = pdns.REFUSED
        return true
    end

    return false
end
```

```ini
# recursor.conf
lua-dns-script=/etc/powerdns/filter.lua
```

## Step 6: Validate and Test

```bash
# Check configuration
pdns_recursor --config-check

# Restart
systemctl restart pdns-recursor
systemctl status pdns-recursor

# Test from IPv6 client
dig AAAA google.com @::1
dig AAAA ipv6.google.com @::1

# Check DNSSEC
dig +dnssec AAAA cloudflare.com @::1
# Look for AD flag

# Statistics
rec_control get all | grep questions
```

## Rate Limiting

```ini
# /etc/powerdns/recursor.conf

# Limit queries per second from a single source
# (protects against abuse from IPv6 clients)
max-qps-ip=100
max-qps=10000

# Throttle sources that cause SERVFAIL
throttle-ip-enable=yes
```

## Conclusion

PowerDNS Recursor listens on IPv6 by adding `::` or specific addresses to `local-address`. DNSSEC validation, Lua scripting, and per-IP rate limiting make it production-ready. Use OneUptime to monitor recursor availability, cache hit rates, and query response times from IPv6 endpoints.
