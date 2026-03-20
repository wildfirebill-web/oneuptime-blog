# How to Debug gRPC IPv6 Connection Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, IPv6, Debugging, Troubleshooting, Networking

Description: Debug common gRPC connection issues over IPv6 including address format errors, TLS mismatches, and DNS resolution failures.

## Common gRPC IPv6 Issues

| Error | Cause |
|-------|-------|
| `name resolution failure` | Missing AAAA record or DNS not resolving IPv6 |
| `connection refused` | Server not listening on IPv6 or wrong port |
| `address format invalid` | Missing brackets around IPv6 in target address |
| `context deadline exceeded` | Firewall blocking gRPC port or IPv6 routing issue |
| `TLS handshake failure` | Certificate doesn't include IPv6 SAN |

## Step 1: Verify IPv6 Connectivity

```bash
# Check basic IPv6 connectivity first
ping6 -c 4 2001:db8::1

# Verify gRPC port is open over IPv6
nc -6 -zv 2001:db8::1 50051

# Check server is listening on IPv6
sudo ss -tlnp | grep 50051
# Look for [::]:50051 or *:50051 in the output
```

## Step 2: Enable gRPC Debug Logging

```bash
# Go: enable verbose gRPC logging
GRPC_GO_LOG_VERBOSITY_LEVEL=99 \
GRPC_GO_LOG_SEVERITY_LEVEL=info \
go run server.go 2>&1 | grep -E "IPv6|addr|resolve"

# Python: enable gRPC debug logging
GRPC_VERBOSITY=DEBUG \
GRPC_TRACE=all \
python server.py 2>&1 | head -50

# Node.js: enable gRPC channel tracing
GRPC_VERBOSITY=DEBUG \
GRPC_TRACE=channel,subchannel,address_sorting \
node server.js
```

## Step 3: Check Address Format

IPv6 addresses in gRPC target strings must be wrapped in brackets:

```go
// Wrong — will fail with "invalid address" or resolve as hostname
conn, err := grpc.NewClient("2001:db8::1:50051", ...)  // WRONG

// Correct — IPv6 with brackets
conn, err := grpc.NewClient("[2001:db8::1]:50051", ...)  // CORRECT

// Also correct — DNS resolution (returns AAAA records)
conn, err := grpc.NewClient("dns:///grpc.example.com:50051", ...)
```

```python
# Python: same bracket requirement
channel = grpc.insecure_channel("2001:db8::1:50051")  # WRONG
channel = grpc.insecure_channel("[2001:db8::1]:50051")  # CORRECT
```

## Step 4: Debug with grpcurl

grpcurl is invaluable for testing gRPC endpoints:

```bash
# Test connectivity to IPv6 server
grpcurl -plaintext '[2001:db8::1]:50051' list

# If this fails, check:
# 1. Server is listening: sudo ss -tlnp | grep 50051
# 2. Firewall: sudo ip6tables -L | grep 50051
# 3. Route: ip -6 route get 2001:db8::1

# Test with DNS (verifies AAAA resolution)
grpcurl -plaintext 'grpc.example.com:50051' list

# Test with TLS
grpcurl -insecure '[2001:db8::1]:443' list
grpcurl -cacert ca.crt '[2001:db8::1]:443' list

# Verbose output
grpcurl -v -plaintext '[2001:db8::1]:50051' helloworld.Greeter/SayHello
```

## Step 5: Certificate Issues with IPv6

TLS certificates for gRPC over IPv6 must include the IPv6 address as a SAN:

```bash
# Check if certificate includes IPv6 SAN
openssl x509 -in server.crt -text | grep -A 5 "Subject Alternative Name"
# Look for: IP Address:2001:DB8::1

# Generate certificate with IPv6 SAN
openssl req -x509 -newkey rsa:4096 \
  -keyout server.key -out server.crt \
  -days 365 -nodes \
  -subj "/CN=example.com" \
  -addext "subjectAltName=DNS:example.com,IP:2001:db8::1,IP:::1"

# Verify SAN includes IPv6
openssl x509 -in server.crt -noout -ext subjectAltName
```

## Step 6: Wireshark Analysis

```bash
# Capture gRPC traffic over IPv6
sudo tcpdump -i eth0 -w grpc-ipv6.pcap "ip6 and tcp port 50051"

# Open in Wireshark — filter for gRPC
# Display filter: grpc or http2

# Or analyze directly
tcpdump -r grpc-ipv6.pcap -n "ip6"
```

## Step 7: Common Firewall Issues

```bash
# Check if IPv6 port 50051 is open
sudo ip6tables -L INPUT -v -n | grep 50051

# If missing, add rule
sudo ip6tables -A INPUT -p tcp --dport 50051 -j ACCEPT

# Also check for explicit REJECT rules
sudo ip6tables -L INPUT -v -n | grep -E "REJECT|DROP"

# Check that IPv6 forwarding is enabled (for container environments)
sysctl net.ipv6.conf.all.forwarding
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) TCP monitors on IPv6 addresses for early detection of gRPC connectivity issues. A failing TCP check on port 50051 indicates either a server crash or firewall block before you even start debugging gRPC-specific issues.

## Conclusion

Most gRPC IPv6 issues stem from missing brackets in address strings, TLS certificates without IPv6 SANs, or firewall rules blocking the gRPC port over IPv6. Always verify basic IPv6 connectivity before debugging gRPC-specific issues, and use grpcurl for rapid endpoint testing.
