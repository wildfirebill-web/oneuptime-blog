# How to Use curl with IPv6 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, curl, HTTP, Network Diagnostics, API Testing, Web

Description: Use curl to make HTTP requests over IPv6, force IPv6 connections, test IPv6 web services, and handle IPv6 addresses in URLs correctly.

## Introduction

`curl` fully supports IPv6 for HTTP, HTTPS, FTP, and other protocols. IPv6 addresses in URLs must be enclosed in square brackets per RFC 2732. The `-6` flag forces IPv6 connections, and `--resolve` allows testing IPv6 endpoints without DNS changes. Understanding curl's IPv6 behavior is essential for testing dual-stack and IPv6-only web services.

## Basic IPv6 curl Commands

```bash
# Connect to an IPv6 address directly (brackets required in URL)

curl http://[2001:db8::1]/

# HTTPS over IPv6
curl https://[2001:db8::1]/

# Force IPv6 for a hostname (don't use IPv4)
curl -6 https://example.com

# Show verbose output including IPv6 address used
curl -v -6 https://example.com 2>&1 | grep "Connected to"

# Show just the IP address curl used
curl -w "%{remote_ip}\n" -o /dev/null -s https://example.com

# Check if a host prefers IPv6
curl -w "Connected via: %{remote_ip}\n" -s -o /dev/null https://example.com
```

## Testing IPv6 Web Services

```bash
# Check HTTP response code over IPv6
curl -6 -o /dev/null -w "%{http_code}" https://example.com

# Get response headers over IPv6
curl -6 -I https://example.com

# Test with specific IPv6 source address (bind interface)
curl --interface eth0 -6 https://example.com

# Test IPv6 connectivity to a specific IP
curl -6 -I https://[2001:db8::1]/ \
    -H "Host: example.com"

# Send POST request over IPv6
curl -6 -X POST \
    -H "Content-Type: application/json" \
    -d '{"key": "value"}' \
    https://[2001:db8::1]/api/endpoint
```

## Using --resolve to Override DNS

```bash
# Test a hostname against its IPv6 address without DNS change
# Format: hostname:port:address
curl --resolve "example.com:443:2001:db8::1" \
    https://example.com/

# Test HTTP over IPv6 with host override
curl --resolve "example.com:80:2001:db8::1" \
    http://example.com/

# Combine with -v to see connection details
curl -v --resolve "example.com:443:2001:db8::1" \
    https://example.com/ 2>&1 | head -20

# Test multiple endpoints
for addr in 2001:db8::1 2001:db8::2 2001:db8::3; do
    echo -n "Testing $addr: "
    code=$(curl -s -o /dev/null -w "%{http_code}" \
        --resolve "lb.example.com:443:$addr" \
        https://lb.example.com/ \
        --max-time 5 2>/dev/null)
    echo "$code"
done
```

## IPv6 Connectivity Testing Script

```bash
#!/bin/bash
# test-ipv6-http.sh

TARGET="${1:-https://ipv6.google.com}"

echo "=== IPv6 HTTP Connectivity Test ==="
echo "Target: $TARGET"

# Test IPv6-forced connection
echo ""
echo "IPv6 forced (-6):"
curl -6 -s -o /dev/null -w "  HTTP: %{http_code}  Time: %{time_total}s  IP: %{remote_ip}" \
    "$TARGET" --max-time 10
echo ""

# Test default (Happy Eyeballs)
echo ""
echo "Default protocol (Happy Eyeballs):"
curl -s -o /dev/null -w "  HTTP: %{http_code}  Time: %{time_total}s  IP: %{remote_ip}" \
    "$TARGET" --max-time 10
echo ""

# Test IPv4 for comparison
echo ""
echo "IPv4 forced (-4):"
curl -4 -s -o /dev/null -w "  HTTP: %{http_code}  Time: %{time_total}s  IP: %{remote_ip}" \
    "$TARGET" --max-time 10
echo ""
```

## Handling Link-Local IPv6 Addresses

```bash
# Link-local addresses require a scope ID
# Use %25 (URL-encoded %) in URLs
curl http://[fe80::1%25eth0]/

# Or use --interface to bind to the right interface
curl --interface eth0 http://[fe80::1]/

# Test with explicit interface specification
curl -6 --interface eth0 http://[fe80::1%25eth0]/path
```

## curl Format for Scripting

```bash
# Full timing breakdown over IPv6
curl -6 -w "\n
  DNS lookup:    %{time_namelookup}s
  Connect:       %{time_connect}s
  TLS handshake: %{time_appconnect}s
  Total:         %{time_total}s
  Remote IP:     %{remote_ip}
  HTTP code:     %{http_code}\n" \
  -s -o /dev/null https://example.com
```

## Conclusion

curl IPv6 support requires square brackets around IPv6 addresses in URLs (`http://[2001:db8::1]/`). Use `-6` to force IPv6 connections for any hostname, and `--resolve` to test a specific IPv6 backend address while using the correct `Host` header. The `-w "%{remote_ip}"` output flag confirms which IP address curl actually connected to, which is essential for verifying Happy Eyeballs behavior.
