# How to Force DNS Queries Over TCP Instead of UDP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, TCP, UDP, Linux, Networking, Configuration, Troubleshooting

Description: Configure DNS clients to use TCP instead of UDP for all queries, useful when UDP is blocked, to verify TCP DNS works, or to bypass UDP packet size limitations.

## Introduction

DNS defaults to UDP for efficiency but falls back to TCP when responses are too large. In some environments, UDP port 53 is blocked while TCP port 53 is allowed, requiring explicit TCP configuration. Forcing TCP is also useful for debugging: if DNS works over TCP but not UDP, you have a UDP firewall issue. This guide covers forcing TCP at the query, application, and system resolver levels.

## Force TCP with dig

```bash
# Force TCP for a single query:

dig example.com +tcp

# Verify TCP is used:
tcpdump -i eth0 -n 'tcp port 53' -c 5 &
dig example.com +tcp
# Should see TCP 3-way handshake followed by DNS query

# Test TCP connectivity without DNS:
nc -z 8.8.8.8 53 && echo "TCP 53 reachable" || echo "TCP 53 blocked"
nc -zu 8.8.8.8 53 && echo "UDP 53 reachable" || echo "UDP 53 blocked"

# Common use case: test from behind a firewall:
dig @8.8.8.8 example.com +tcp
# If this works but UDP doesn't: UDP 53 is blocked
```

## Force TCP in systemd-resolved

```bash
# Enable DNS over TLS (which uses TCP with TLS):
# /etc/systemd/resolved.conf:
cat >> /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=8.8.8.8
DNSOverTLS=yes   # Force DoT (TCP/853)
EOF

systemctl restart systemd-resolved

# Verify DoT is being used:
resolvectl status | grep -i "tls\|over"

# For plain TCP (not TLS), systemd-resolved doesn't have a direct TCP-only option
# Use Unbound or direct application configuration instead
```

## Force TCP in Unbound

```bash
# Configure Unbound to use TCP for upstream queries:
cat >> /etc/unbound/unbound.conf << 'EOF'
server:
    # Force TCP for all upstream queries:
    tcp-upstream: yes
EOF

unbound-control reload

# Verify Unbound is using TCP:
tcpdump -i eth0 -n 'tcp port 53' &
dig @127.0.0.1 -p 5335 google.com
# Should see TCP connections from Unbound to upstream resolver
```

## Application-Level TCP DNS

```python
#!/usr/bin/env python3
# Force TCP in Python DNS query (using dnspython):
# pip install dnspython

import dns.resolver
import dns.query
import dns.message

# Method 1: Use dnspython's TCP-aware functions
def query_dns_tcp(domain, record_type='A', server='8.8.8.8'):
    qname = dns.name.from_text(domain)
    q = dns.message.make_query(qname, record_type)

    # Force TCP:
    response = dns.query.tcp(q, server, port=53, timeout=5)
    return response

response = query_dns_tcp('example.com')
for rr in response.answer:
    print(rr)

# Method 2: Resolver with TCP flag:
resolver = dns.resolver.Resolver()
resolver.nameservers = ['8.8.8.8']
# dnspython handles TCP fallback automatically when TC bit is set
# To always use TCP, send initial query with truncation flag set
# This forces immediate retry over TCP
```

## Configure Resolver for TCP (BIND9)

```bash
# If using BIND as a caching resolver, force TCP for upstream queries:
# /etc/bind/named.conf.options:
options {
    # ... other options ...

    # Try TCP after UDP fails:
    # (BIND handles this automatically)

    # But you can also use query-source to limit to TCP:
    # Note: BIND doesn't have a "tcp-only" option for upstream queries
    # Use forward-only with a specific TCP resolver instead:
};

# Alternative: Use BIND to forward to a DoT resolver:
options {
    forward only;
    forwarders {
        8.8.8.8 port 853 tls GOOGLE;
    };
};
```

## Verify TCP DNS is Working

```bash
# Comprehensive test: verify TCP 53 works and UDP 53 works:
echo "=== UDP DNS Test ==="
dig @8.8.8.8 google.com | grep "Query time"

echo "=== TCP DNS Test ==="
dig @8.8.8.8 google.com +tcp | grep "Query time"

# Test specific port:
# DNS over TLS (port 853):
dig @8.8.8.8 google.com -p 853 +tcp +tls

# Capture to verify transport:
tcpdump -i eth0 -n '(tcp or udp) and port 53' -c 10 &
dig @8.8.8.8 google.com        # UDP
dig @8.8.8.8 google.com +tcp   # TCP
wait
```

## Conclusion

Force TCP for DNS with `dig +tcp` for single queries. When UDP is blocked, TCP DNS provides a reliable fallback - verify both with `nc -z` (TCP) and `nc -zu` (UDP) tests. For production environments where UDP is blocked, configure `systemd-resolved` with `DNSOverTLS=yes` for encrypted TCP, or configure Unbound with `tcp-upstream: yes` for unencrypted TCP. DNS over TCP is 1-2ms slower than UDP due to the handshake but functionally identical for the responses themselves.
