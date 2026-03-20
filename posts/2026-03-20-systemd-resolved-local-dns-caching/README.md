# How to Use systemd-resolved for Local DNS Caching

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Systemd-resolved, Caching, Linux, Performance, Configuration

Description: Configure and monitor systemd-resolved as a local DNS cache to reduce DNS latency, improve reliability, and handle DNS over TLS on modern Linux systems.

## Introduction

`systemd-resolved` is a local DNS stub resolver and cache included with systemd. On modern Linux distributions (Ubuntu 17.10+, Fedora 33+), it runs by default, caching DNS responses to improve performance. Understanding how to configure, monitor, and optimize it improves DNS reliability and reduces latency for all applications on the system.

## Verify systemd-resolved is Active

```bash
# Check if it's running:

systemctl status systemd-resolved

# Check if /etc/resolv.conf points to it:
cat /etc/resolv.conf
# Should contain: nameserver 127.0.0.53
# Or be a symlink to: /run/systemd/resolve/stub-resolv.conf

# Check the resolved stub listener:
ss -ulnp | grep 53
# Should show: 127.0.0.53:53 (bound by systemd-resolved)
```

## View Current Configuration

```bash
# Show complete DNS configuration per interface:
resolvectl status
# Shows:
# - Current DNS server per interface
# - DNS search domains
# - DNS over TLS status
# - DNSSEC validation

# Show only global settings:
resolvectl status | grep -A10 "Global"

# Show per-interface settings:
resolvectl status eth0
```

## Configure DNS Servers

```bash
# /etc/systemd/resolved.conf:
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
# Upstream DNS servers:
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9 208.67.222.222

# DNS search domains:
Domains=~.    # Use for all domains (global fallback)

# DNS over TLS:
DNSOverTLS=opportunistic   # Use TLS if available, plain DNS as fallback
# Options: no, opportunistic, yes

# DNSSEC:
DNSSEC=allow-downgrade   # Validate if available
# Options: no, allow-downgrade, yes

# Cache settings:
Cache=yes                  # Enable caching (default)
CacheFromLocalhost=no      # Don't cache queries from loopback

# Stub listener:
DNSStubListener=yes        # Listen on 127.0.0.53:53
EOF

systemctl restart systemd-resolved
```

## Monitor Cache Performance

```bash
# View cache statistics:
resolvectl statistics
# Shows:
# Current Cache Size: N        (entries currently in cache)
# Cache Hits: N               (queries answered from cache)
# Cache Misses: N             (queries that went to upstream)

# Cache hit rate calculation:
resolvectl statistics | python3 -c "
import sys, re
data = sys.stdin.read()
hits = int(re.search(r'Cache Hits:\s+(\d+)', data).group(1))
misses = int(re.search(r'Cache Misses:\s+(\d+)', data).group(1))
total = hits + misses
if total > 0:
    print(f'Cache hit rate: {hits/total*100:.1f}% ({hits}/{total})')
else:
    print('No queries recorded')
"
```

## Flush and Test Cache

```bash
# Flush all caches:
resolvectl flush-caches

# Test cache performance:
# First query (uncached):
time resolvectl query google.com

# Second query (cached):
time resolvectl query google.com
# Should be significantly faster on second query

# Watch cache fill:
resolvectl statistics | grep "Current Cache Size"
for i in $(seq 1 10); do
    dig $(cat /dev/urandom | tr -dc 'a-z' | head -c 5).example.com > /dev/null 2>&1
done
resolvectl statistics | grep "Current Cache Size"
```

## Per-Interface DNS Configuration

```bash
# Set different DNS servers for different interfaces:
# (These override global settings for that interface)

# Set DNS for a specific interface:
resolvectl dns eth0 10.20.0.1 10.20.0.2

# Set search domain for an interface:
resolvectl domain eth0 company.internal internal

# These changes are temporary (lost on restart)
# For permanent per-interface config, use /etc/systemd/network/:
cat > /etc/systemd/network/10-eth0.network << 'EOF'
[Match]
Name=eth0

[Network]
DNS=10.20.0.1
DNS=10.20.0.2
Domains=company.internal internal

[Address]
Address=10.20.0.50/24
EOF
```

## Disable for Specific Domains

```bash
# Route specific domains to specific DNS servers:
# Useful for split-horizon: internal domains to internal DNS

resolvectl domain eth0 ~company.internal  # Route .company.internal to eth0's DNS
resolvectl dns eth0 10.20.0.1

# Or in /etc/systemd/resolved.conf:
# Domains=~company.internal   # With ~ means "route this domain to this server"
```

## Conclusion

`systemd-resolved` provides automatic DNS caching for all applications on the system. Monitor cache performance with `resolvectl statistics` - a hit rate above 80% means your TTLs and cache size are appropriate. Configure upstream DNS servers in `/etc/systemd/resolved.conf` with `DNSOverTLS=opportunistic` for privacy. For split-horizon DNS, route internal domains to internal DNS servers using `resolvectl domain interface ~internal.domain`. The `resolvectl flush-caches` command is the correct way to force fresh DNS lookups after record changes.
