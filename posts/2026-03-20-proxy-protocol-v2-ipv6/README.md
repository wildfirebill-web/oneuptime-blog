# How to Configure PROXY Protocol v2 for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, PROXY Protocol, HAProxy, Nginx, Load Balancer, Client IP

Description: Configure PROXY Protocol v2 to carry IPv6 client addresses through load balancers and reverse proxies, enabling backends to see the original client IPv6 address.

## Introduction

PROXY Protocol is a network protocol that prepends a header to TCP connections carrying the original client IP address and port. Version 2 uses a binary format and supports IPv6 addresses (16 bytes) natively. It is used by HAProxy, Nginx, AWS ELB, and other load balancers to pass the real client IP to backends without relying on HTTP headers.

## How PROXY Protocol v2 Works

```
Client (2001:db8::1:1234) → Load Balancer → Backend (with PROXY Protocol header)

Binary header at start of TCP stream:
  Signature: \x0D\x0A\x0D\x0A\x00\x0D\x0A\x51\x55\x49\x54\x0A
  Version/Command: 0x21 (v2, PROXY)
  Family/Protocol: 0x21 (AF_INET6, STREAM)
  Length: 0x0024 (36 bytes for IPv6+ports)
  Source Address: 2001:db8::1 (16 bytes)
  Destination Address: 2001:db8::2 (16 bytes)
  Source Port: 1234 (2 bytes)
  Destination Port: 443 (2 bytes)
```

## HAProxy: Send PROXY Protocol v2

```haproxy
# /etc/haproxy/haproxy.cfg

frontend web_ipv6
    bind [::]:443 ssl crt /etc/ssl/certs/app.pem
    bind 0.0.0.0:443 ssl crt /etc/ssl/certs/app.pem

    default_backend app_servers

backend app_servers
    # Send PROXY Protocol v2 header to backends
    server app1 10.0.0.1:8080 send-proxy-v2
    server app2 10.0.0.2:8080 send-proxy-v2
    server app3 [2001:db8::10]:8080 send-proxy-v2
```

## Nginx: Accept PROXY Protocol v2

```nginx
# /etc/nginx/conf.d/proxy-protocol.conf

server {
    # Accept PROXY Protocol v2 on this port
    listen 8080 proxy_protocol;
    listen [::]:8080 proxy_protocol;

    # Real client IP is now available from proxy_protocol header
    # $proxy_protocol_addr contains the IPv6 address from the header
    set_real_ip_from 10.0.0.0/8;
    set_real_ip_from fd00::/8;
    real_ip_header proxy_protocol;

    location / {
        proxy_pass http://backend:9090;

        # Pass the original client IPv6 address downstream
        proxy_set_header X-Forwarded-For $proxy_protocol_addr;
        proxy_set_header X-Real-IP       $proxy_protocol_addr;

        # Log shows real client IPv6 address
        # access_log uses $proxy_protocol_addr
    }

    # Include client IPv6 in access log
    log_format proxy_proto '$proxy_protocol_addr - $remote_user [$time_local] '
                           '"$request" $status $body_bytes_sent';
    access_log /var/log/nginx/access.log proxy_proto;
}
```

## HAProxy: Receive PROXY Protocol v2 and Log IPv6 Client

```haproxy
frontend internal
    # Accept PROXY Protocol v2 from upstream load balancer
    bind [::]:8080 accept-proxy

    # %[fc_pp_authority] gives the client IP from PROXY Protocol header
    log-format "%ci:%cp [%t] %ft %b/%s %Tw/%Tc/%Tt %B %ts"
    # %ci = client IP (from PROXY Protocol if accept-proxy is set)

    default_backend app
```

## Application: Parse PROXY Protocol v2 in Python

```python
#!/usr/bin/env python3
# proxy_protocol_v2_parser.py

import socket
import struct
import ipaddress

PROXY_V2_SIGNATURE = b'\x0D\x0A\x0D\x0A\x00\x0D\x0A\x51\x55\x49\x54\x0A'
PROXY_V2_HEADER_LEN = 16

def parse_proxy_v2_header(data: bytes) -> dict:
    """Parse PROXY Protocol v2 binary header from TCP stream."""
    if len(data) < PROXY_V2_HEADER_LEN:
        raise ValueError("Header too short")

    if data[:12] != PROXY_V2_SIGNATURE:
        raise ValueError("Not a PROXY Protocol v2 header")

    ver_cmd = data[12]
    fam_proto = data[13]
    length = struct.unpack('!H', data[14:16])[0]

    version = (ver_cmd >> 4) & 0x0F
    command = ver_cmd & 0x0F

    address_family = (fam_proto >> 4) & 0x0F
    protocol = fam_proto & 0x0F

    # AF_INET6 = 2
    if address_family == 2 and length >= 36:
        payload = data[16:16 + length]
        src_addr = ipaddress.ip_address(payload[0:16])
        dst_addr = ipaddress.ip_address(payload[16:32])
        src_port = struct.unpack('!H', payload[32:34])[0]
        dst_port = struct.unpack('!H', payload[34:36])[0]
        return {
            "version": version,
            "address_family": "IPv6",
            "src_addr": str(src_addr),
            "dst_addr": str(dst_addr),
            "src_port": src_port,
            "dst_port": dst_port,
            "header_length": PROXY_V2_HEADER_LEN + length,
        }

    raise ValueError(f"Unsupported address family: {address_family}")

def handle_connection(conn: socket.socket) -> None:
    """Handle an incoming connection with PROXY Protocol v2."""
    # Read enough data to parse the header
    data = conn.recv(1024)

    try:
        info = parse_proxy_v2_header(data)
        client_ipv6 = info["src_addr"]
        client_port = info["src_port"]
        print(f"Real client: [{client_ipv6}]:{client_port}")

        # Process the rest of the data after the PROXY Protocol header
        payload = data[info["header_length"]:]
        # ... handle payload
    except ValueError as e:
        print(f"No PROXY Protocol header: {e}")
        payload = data
        # ... handle payload without real IP
```

## Testing PROXY Protocol v2

```bash
# Test with haproxy sending PROXY Protocol v2 to local port
# Use haproxy in client mode or the pp2 tool:

# Install proxychains or pp2 test client
# Send a test connection with PROXY v2 header carrying IPv6 source

# Alternatively, test with socat:
# socat accepts PROXY Protocol v2 and forwards
socat -v TCP6-LISTEN:8081,fork \
    PROXY2:backend:8080,proxyport=8081,socktype=1

# Check Nginx accepts PROXY Protocol:
curl --haproxy-protocol -6 http://[::1]:8080/
```

## Common Pitfalls

| Issue | Cause | Fix |
|-------|-------|-----|
| Connection refused after enabling | Backend doesn't accept PROXY Protocol | Enable `proxy_protocol` on listener |
| `$proxy_protocol_addr` is empty | `real_ip_header proxy_protocol` not set | Add `real_ip_header proxy_protocol` |
| HAProxy shows old client IP | Backend using `$remote_addr` | Switch to `$proxy_protocol_addr` |
| Binary garbage in HTTP logs | PROXY v2 header not stripped | Use `proxy_protocol` listen flag |

## Conclusion

PROXY Protocol v2 provides a reliable, binary-format mechanism for carrying IPv6 client addresses through load balancers without depending on HTTP headers. HAProxy sends v2 headers with `send-proxy-v2` and accepts them with `accept-proxy`. Nginx accepts them with the `proxy_protocol` listen parameter and exposes the client address via `$proxy_protocol_addr`. This approach works for any TCP-based protocol, not just HTTP, making it valuable for databases, SMTP relays, and other services where HTTP headers are unavailable.
