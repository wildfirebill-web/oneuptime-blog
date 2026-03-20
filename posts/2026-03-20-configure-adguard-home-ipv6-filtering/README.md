# How to Configure AdGuard Home for IPv6 DNS Filtering

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AdGuard Home, DNS, IPv6, Filtering, DoH, DoT, Ad Blocking

Description: Configure AdGuard Home to listen on IPv6, serve encrypted DNS (DoH/DoT) over IPv6, and filter ads and trackers for dual-stack and IPv6-only clients.

## Introduction

AdGuard Home is a self-hosted DNS ad-blocker with a web UI, support for DNS-over-HTTPS, DNS-over-TLS, and DNS-over-QUIC. It can listen on IPv6 and route IPv6 clients through filtering upstream resolvers.

## Installation

```bash
# Download and install

curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

# Or via Docker
docker run -d \
    --name adguardhome \
    -v /opt/adguardhome/work:/opt/adguardhome/work \
    -v /opt/adguardhome/conf:/opt/adguardhome/conf \
    -p 53:53/udp -p 53:53/tcp \
    -p 3000:3000/tcp \
    adguard/adguardhome
```

## Step 1: Configure IPv6 Listening

```yaml
# /opt/AdGuardHome/AdGuardHome.yaml

dns:
  # Bind to all IPv6 and IPv4 interfaces
  bind_hosts:
    - "::"
    - "0.0.0.0"
  port: 53

  # Or specific IPv6 address
  # bind_hosts:
  #   - "2001:db8::1"
  #   - "127.0.0.1"
```

## Step 2: IPv6 Upstream Resolvers

```yaml
# /opt/AdGuardHome/AdGuardHome.yaml

dns:
  upstream_dns:
    - "https://dns.cloudflare.com/dns-query"        # DoH
    - "tls://1dot1dot1dot1.cloudflare-dns.com"      # DoT
    - "[2606:4700:4700::1111]"                       # Plain IPv6
    - "[2606:4700:4700::1001]"
    - "8.8.8.8"                                      # IPv4 fallback

  # Use parallel queries for speed
  all_servers: true

  # Bootstrap resolvers (to resolve DoH hostnames)
  bootstrap_dns:
    - "2606:4700:4700::1111"
    - "8.8.8.8"
```

## Step 3: Enable DoH and DoT on IPv6

```yaml
# /opt/AdGuardHome/AdGuardHome.yaml

tls:
  enabled: true
  server_name: "dns.example.com"
  force_https: false
  port_https: 443       # DoH
  port_dns_over_tls: 853  # DoT
  port_dns_over_quic: 784  # DoQ

  # Certificate (Let's Encrypt or self-signed)
  certificate_path: /etc/ssl/dns.example.com.crt
  private_key_path: /etc/ssl/dns.example.com.key
```

```bash
# Access DoH over IPv6:
# https://[2001:db8::1]/dns-query

# Access DoT over IPv6:
# tls://[2001:db8::1]:853

# Test DoH
curl -s "https://[2001:db8::1]/dns-query?name=example.com&type=AAAA" \
    -H "Accept: application/dns-json"
```

## Step 4: Filtering Lists

```yaml
# /opt/AdGuardHome/AdGuardHome.yaml

filters:
  - enabled: true
    url: "https://adguardteam.github.io/AdGuardSDNSFilter/Filters/filter.txt"
    name: AdGuard DNS filter
    id: 1
  - enabled: true
    url: "https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts"
    name: StevenBlack Hosts
    id: 2
```

## Step 5: Local AAAA Records

```yaml
# /opt/AdGuardHome/AdGuardHome.yaml

# Custom DNS rewrites (local AAAA)
rewrites:
  - domain: "homeserver.local"
    answer: "2001:db8::50"
  - domain: "nas.local"
    answer: "2001:db8::51"
  - domain: "*.internal"
    answer: "2001:db8::1"
```

## Step 6: Test

```bash
# Restart AdGuard Home
systemctl restart AdGuardHome

# Test AAAA query filtering
dig AAAA doubleclick.net @2001:db8::1
# Returns: 0.0.0.0 or :: (blocked)

# Test local record
dig AAAA homeserver.local @2001:db8::1

# Check web UI
curl http://[2001:db8::1]:3000
```

## Conclusion

AdGuard Home supports IPv6 listening and upstream resolution natively. Set `bind_hosts: ["::"]` and add IPv6 upstream resolvers to handle both IPv4-mapped and native IPv6 clients. The DoH/DoT endpoints work over IPv6 connections. Monitor AdGuard Home query throughput and filter hit rates with OneUptime.
