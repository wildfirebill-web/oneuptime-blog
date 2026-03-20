# How to Troubleshoot Proxy IPv6 Connection Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Proxy, Troubleshooting, Networking, curl, Debugging

Description: A systematic guide to diagnosing and resolving IPv6 proxy connection failures, covering common errors, diagnostic tools, and step-by-step troubleshooting techniques.

---

IPv6 proxy connection failures often stem from misconfigured address formatting, firewall rules blocking IPv6, or proxy servers that aren't listening on their IPv6 interfaces. This guide walks through a structured troubleshooting approach.

## Step 1: Verify Basic IPv6 Connectivity

Before blaming the proxy, confirm IPv6 connectivity exists on the host:

```bash
# Check if IPv6 is enabled and addresses are assigned

ip -6 addr show

# Test basic IPv6 reachability to a known host
ping6 2001:4860:4860::8888

# Trace the path to the proxy server
traceroute6 2001:db8::1
```

## Step 2: Confirm the Proxy is Listening on IPv6

A common issue is the proxy only binds to IPv4. Check with:

```bash
# On the proxy server, check what interfaces the proxy listens on
ss -tlnp | grep 3128

# Expected output for dual-stack listening:
# LISTEN 0 128 *:3128 *:*   (all interfaces)
# Or for IPv6-only:
# LISTEN 0 128 [::]:3128 [::]:*
```

If the proxy only shows `0.0.0.0:3128`, it's not listening on IPv6. Update the proxy config (e.g., Squid):

```bash
# /etc/squid/squid.conf
# Listen on both IPv4 and IPv6
http_port 3128

# Or explicitly on IPv6 only
http_port [::]:3128
```

## Step 3: Test Direct Proxy Connection

Use curl with verbose output to see exactly where the connection fails:

```bash
# Test with maximum verbosity - shows SSL handshake and connection details
curl -v --proxy "http://[2001:db8::1]:3128" \
     https://httpbin.org/ip 2>&1

# Force IPv6 for the connection to the proxy itself
curl -6 -v --proxy "http://[2001:db8::1]:3128" \
     https://httpbin.org/ip 2>&1
```

## Step 4: Check Firewall Rules

IPv6 has its own firewall chain (`ip6tables`) that is often misconfigured:

```bash
# View IPv6 firewall rules
ip6tables -L -n -v

# Check if proxy port is allowed for IPv6
ip6tables -L INPUT -n | grep 3128

# Temporarily allow proxy port (test only)
ip6tables -A INPUT -p tcp --dport 3128 -j ACCEPT
ip6tables -A INPUT -p tcp --sport 3128 -j ACCEPT
```

## Step 5: Diagnose DNS Resolution Issues

Proxy issues often originate from DNS not returning IPv6 addresses:

```bash
# Check if your DNS server returns AAAA records
dig AAAA proxy.example.com

# Use an IPv6-capable DNS server explicitly
dig AAAA proxy.example.com @2001:4860:4860::8888

# Test if the application resolves the proxy hostname to IPv6
python3 -c "import socket; print(socket.getaddrinfo('proxy.example.com', 3128))"
```

## Step 6: Analyze with tcpdump

Capture IPv6 proxy traffic to see what's happening at the packet level:

```bash
# Capture all IPv6 TCP traffic to/from proxy port
tcpdump -i eth0 -n ip6 and tcp port 3128

# Save capture for analysis
tcpdump -i eth0 -w /tmp/proxy_capture.pcap ip6 and tcp port 3128

# Read the capture
tcpdump -r /tmp/proxy_capture.pcap -nn
```

## Step 7: Test Application-Level Proxy Parsing

Some applications misparse IPv6 proxy URLs. Validate your proxy URL format:

```python
from urllib.parse import urlparse

# Verify the proxy URL parses correctly
proxy_url = "http://[2001:db8::1]:3128"
parsed = urlparse(proxy_url)

print(f"Scheme: {parsed.scheme}")    # Should be 'http'
print(f"Hostname: {parsed.hostname}") # Should be '2001:db8::1' (no brackets)
print(f"Port: {parsed.port}")         # Should be 3128
```

## Common Error Messages and Solutions

| Error | Likely Cause | Solution |
|-------|-------------|----------|
| `Connection refused` | Proxy not listening on IPv6 | Enable IPv6 listening in proxy config |
| `Network unreachable` | No IPv6 route | Check routing table with `ip -6 route` |
| `Name resolution failed` | DNS returns no AAAA | Configure DNS or use IP directly |
| `SSL certificate error` | SAN mismatch for IPv6 | Regenerate cert with IPv6 SAN |
| `407 Proxy Authentication` | Auth not sent | Include credentials in proxy URL |

## Monitoring Proxy Connection Health

Use a simple script to continuously check proxy health over IPv6:

```bash
#!/bin/bash
# Check proxy health every 30 seconds
PROXY="http://[2001:db8::1]:3128"
TARGET="https://httpbin.org/ip"

while true; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
           --proxy "$PROXY" --max-time 10 "$TARGET")
  TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  echo "$TIMESTAMP - HTTP Status: $STATUS"
  sleep 30
done
```

Systematic diagnosis - from basic connectivity to application-level URL parsing - will resolve the majority of IPv6 proxy connection failures efficiently.
