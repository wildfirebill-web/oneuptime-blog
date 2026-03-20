# How to Configure DNS-over-HTTPS (DoH) with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, DNS-over-HTTPS, DoH, Privacy

Description: A guide to configuring DNS-over-HTTPS (DoH) servers and clients with IPv6 support, enabling encrypted DNS queries over HTTPS using IPv6 transport.

## What Is DNS-over-HTTPS?

DNS-over-HTTPS (DoH), defined in RFC 8484, sends DNS queries inside HTTPS requests to port 443. It provides DNS privacy and security by encrypting queries. DoH can use both IPv4 and IPv6 transport — the DNS query is carried inside HTTPS, which runs over TCP/TLS, which can use IPv6.

## Setting Up a DoH Server with dnsdist over IPv6

dnsdist is a DNS load balancer that supports DoH and IPv6:

```bash
# Install dnsdist
apt install dnsdist

# /etc/dnsdist/dnsdist.conf
```

```lua
-- dnsdist configuration for DoH over IPv6

-- Accept DoH connections on all IPv6 addresses, port 443
-- Requires TLS certificate
addDOHLocal("[::]:443", "/etc/ssl/certs/server.crt", "/etc/ssl/private/server.key",
    "/dns-query", {
        -- CORS header for browser compatibility
        customResponseHeaders = {
            ["access-control-allow-origin"] = "*"
        }
    }
)

-- Also accept standard DNS over IPv6
addLocal("[::]:53")
addLocal("0.0.0.0:53")

-- Send queries to upstream resolvers
newServer({address="2001:4860:4860::8888", name="google-v6"})
newServer({address="8.8.8.8", name="google-v4"})

-- Allow all clients
setACL({"0.0.0.0/0", "::/0"})
```

## Setting Up a DoH Server with CoreDNS over IPv6

```corefile
# /etc/coredns/Corefile

.:443 {
    # Enable DoH - listen on all interfaces including IPv6
    # CoreDNS binds to :: by default for IPv6

    tls /etc/ssl/certs/server.crt /etc/ssl/private/server.key

    # Forward queries to upstream (using IPv6)
    forward . 2001:4860:4860::8888 2001:4860:4860::8844 8.8.8.8 {
        prefer_udp
    }

    cache 300
    log
    errors
}
```

## Configuring Clients to Use DoH over IPv6

### Firefox

1. Open **Settings** → **Network Settings** → **Settings**
2. Enable **DNS over HTTPS**
3. Enter the DoH server URL with IPv6: `https://[2001:db8::doh-server]/dns-query`

Note: Firefox supports IPv6 DoH server addresses using the bracket notation.

### curl with DoH

```bash
# Use curl to make a DoH query over IPv6
# The DoH server must have both A and AAAA records (or use --resolve)
curl -v -H "Content-Type: application/dns-message" \
    --http2 \
    -6 \
    "https://dns.google/dns-query?name=example.com&type=AAAA"

# DoH over IPv6 using POST method
# First encode the DNS query (using the 'kdig' tool or Python)
python3 -c "
import struct, base64

# Build a minimal DNS AAAA query for example.com
# This is a simplified example - production uses proper DNS wire format
query = b'\\x00\\x01\\x01\\x00\\x00\\x01\\x00\\x00\\x00\\x00\\x00\\x00'
query += b'\\x07example\\x03com\\x00\\x00\\x1c\\x00\\x01'
print(base64.b64encode(query).decode())
"
```

### Unbound as DoH Proxy

Unbound can forward to DoH upstreams:

```yaml
# /etc/unbound/unbound.conf

server:
    interface: ::0
    interface: 0.0.0.0
    do-ip6: yes
    access-control: ::/0 allow

# Forward to DoH providers (Unbound supports DoH in 1.17+)
forward-zone:
    name: "."
    # Google DoH
    forward-addr: 2001:4860:4860::8888@853#dns.google
    forward-tls-upstream: yes
```

## Testing DoH over IPv6

```bash
# Test DoH endpoint with curl over IPv6
curl -6 -v "https://[2001:4860:4860::8888]/dns-query?name=example.com&type=AAAA" \
    -H "Accept: application/dns-json"

# Using kdig (from knot-dnsutils)
kdig -6 @2001:4860:4860::8888 +https AAAA example.com

# Using dig with DoH (dig 9.18+)
dig AAAA example.com @https://dns.google/dns-query
```

## Firewall Rules for DoH over IPv6

```bash
# Allow HTTPS (port 443) over IPv6 for DoH
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT
ip6tables -A OUTPUT -p tcp --dport 443 -j ACCEPT

# Allow established connections
ip6tables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

## DoH Server Discovery with SVCB/HTTPS Records

RFC 9462 defines DNS service discovery using SVCB/HTTPS records, enabling automatic DoH discovery:

```dns
; Advertise DoH server via HTTPS record
_dns.example.com.  HTTPS 1 . alpn="h2,h3" ipv6hint="2001:db8::53" port=443
```

## Summary

DNS-over-HTTPS with IPv6 uses standard HTTPS over IPv6 transport. Configure DoH servers (dnsdist or CoreDNS) to listen on `[::]:443` with TLS certificates. Clients connect to `https://[2001:db8::server]/dns-query` using bracket notation for IPv6 addresses in URLs. Test with `curl -6` against your DoH endpoint and ensure port 443/TCP is allowed through IPv6 firewalls.
