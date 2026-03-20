# How to Set Up a Recursive DNS64 Resolver with Unbound

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS64, Unbound, IPv6, NAT64, RFC 6147, Synthesis, Resolver

Description: Configure Unbound as a DNS64 recursive resolver to synthesize AAAA records for IPv6-only clients using NAT64, including custom prefix support.

## Introduction

Unbound supports DNS64 natively. When an IPv6-only client queries for a domain that only has A records, Unbound synthesizes a AAAA record by combining the NAT64 prefix with the IPv4 address, enabling the client to connect through a NAT64 gateway.

## Step 1: Configure Unbound for DNS64

```yaml
# /etc/unbound/unbound.conf

server:
    interface: ::0
    interface: 0.0.0.0
    access-control: ::/0 allow
    access-control: 0.0.0.0/0 allow

    prefer-ip6: yes
    auto-trust-anchor-file: "/var/lib/unbound/root.key"

module-config: "dns64 validator iterator"

dns64:
    # Well-known NAT64 prefix
    prefix: 64:ff9b::/96

    # Synthesize only when no AAAA record exists
    # (default behavior - no extra config needed)
```

## Step 2: Custom NAT64 Prefix

```yaml
# If your NAT64 gateway uses a custom /96 prefix

dns64:
    prefix: 2001:db8:1::/96

# The synthesized address will be:
# 2001:db8:1::<IPv4-address>
# Example: for 1.2.3.4 → 2001:db8:1::102:304
```

## Step 3: Exclude Private IPv4 Ranges

```yaml
# Prevent synthesis for RFC 1918 addresses
# Unbound dns64 module excludes loopback by default

# To handle this, use a local-zone override:
server:
    local-zone: "0.10.in-addr.arpa." nodefault
    local-zone: "168.192.in-addr.arpa." nodefault

# Or use the dns64-synthall option (synthesize even with AAAA)
# dns64:
#     synthall: no  # default: no - don't synthesize if AAAA exists
```

## Step 4: Validate and Test

```bash
# Check configuration
unbound-checkconf

# Initialize trust anchor
unbound-anchor -a /var/lib/unbound/root.key

# Start Unbound
systemctl restart unbound

# Test synthesis - domain with A only
dig AAAA ipv4only.example.com @::1
# Expected: 64:ff9b::<ipv4>

# Test no synthesis for domains with AAAA
dig AAAA google.com @::1
# Expected: real 2607:f8b0:... address (no synthesis)

# Test synthesis of well-known test address
dig AAAA ipv4only.arpa @::1
# 64:ff9b::c000:0200 (192.0.2.0)
```

## Step 5: Integration Test with NAT64

```bash
# From an IPv6-only host using this Unbound as DNS64
export DNS_SERVER="2001:db8::53"

# Check name resolution
host -t AAAA www.example.com $DNS_SERVER

# Test connectivity through NAT64
curl --dns-servers $DNS_SERVER -6 http://ipv4only-site.com/

# Packet capture to verify AAAA synthesis
tcpdump -i eth0 -n 'udp port 53' &
dig AAAA ipv4only.example.com @$DNS_SERVER
```

## Step 6: DNS64 + DNSSEC

```yaml
# Important: DNSSEC cannot validate synthesized records
# Disable validation for DNS64-synthesized responses

server:
    module-config: "dns64 validator iterator"
    # dns64 module runs before validator
    # Synthesized records bypass DNSSEC validation (by design)
    # Real AAAA records are still validated
```

## Monitoring

```bash
# Check Unbound DNS64 statistics
unbound-control stats | grep dns64

# Log synthesis events
# /etc/unbound/unbound.conf
# server:
#   verbosity: 3  (shows dns64 synthesis in logs)
```

## Conclusion

Unbound's DNS64 module is enabled with `module-config: "dns64 validator iterator"` and a `dns64: prefix:` block. The module transparently synthesizes AAAA records for A-only domains, enabling NAT64 connectivity for IPv6-only clients. Monitor synthesis rates alongside NAT64 translation success using OneUptime synthetic checks.
