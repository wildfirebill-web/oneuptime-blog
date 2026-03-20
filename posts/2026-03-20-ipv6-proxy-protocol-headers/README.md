# How to Handle IPv6 in Proxy Protocol Headers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Proxy Protocol, HAProxy, Nginx, Load Balancer, Networking

Description: Configure and parse PROXY Protocol v1 and v2 headers with IPv6 addresses in HAProxy and Nginx to preserve real client IPs through TCP load balancers.

## Introduction

The PROXY Protocol (designed by HAProxy) provides a standardized way to pass connection metadata including the real client IP through TCP load balancers. Unlike HTTP X-Forwarded-For, PROXY Protocol works at the TCP layer and supports IPv6 in both its text-based v1 and binary v2 formats.

## PROXY Protocol Format

### Version 1 (Text)

```text
PROXY TCP4 <src-ip> <dst-ip> <src-port> <dst-port>\r\n
PROXY TCP6 <src-ipv6> <dst-ipv6> <src-port> <dst-port>\r\n

Examples:
PROXY TCP4 192.168.1.1 10.0.0.1 12345 80\r\n
PROXY TCP6 2001:db8::1 2001:db8::10 54321 443\r\n
PROXY UNKNOWN\r\n
```

### Version 2 (Binary)

PROXY Protocol v2 uses a 16-byte signature followed by address data in binary format, with separate address families for IPv4 (`0x11`) and IPv6 (`0x21`).

## HAProxy: Sending PROXY Protocol to Backend

Configure HAProxy to send PROXY Protocol headers with IPv6 support:

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log stdout format raw local0 info

defaults
    log     global
    mode    tcp
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend ipv6_frontend
    bind [::]:443 ssl crt /etc/ssl/certs/example.pem
    bind 0.0.0.0:443 ssl crt /etc/ssl/certs/example.pem
    default_backend app_backend

backend app_backend
    # Send PROXY Protocol v2 to backend (supports IPv6)
    server app1 10.0.0.10:8080 send-proxy-v2

    # Or send PROXY Protocol v1 (text format)
    # server app1 10.0.0.10:8080 send-proxy
```

## Nginx: Receiving PROXY Protocol with IPv6

Configure Nginx to accept PROXY Protocol v1/v2 and extract the real client IP:

```nginx
# /etc/nginx/nginx.conf

stream {
    server {
        listen 8080 proxy_protocol;

        # Log real client IP from PROXY Protocol
        set_real_ip_from 10.0.0.0/8;    # Trust HAProxy IPv4
        set_real_ip_from 2001:db8::/32;  # Trust HAProxy IPv6

        proxy_pass backend;
    }
}

http {
    server {
        listen 8080 proxy_protocol;

        # Trust the load balancer IPs
        set_real_ip_from 10.0.0.0/8;
        set_real_ip_from 2001:db8:proxy::/48;
        real_ip_header proxy_protocol;

        location / {
            # $proxy_protocol_addr contains the real client IP (IPv4 or IPv6)
            add_header X-Real-Client-IP $proxy_protocol_addr;
            proxy_pass http://backend;
        }
    }
}
```

## Parsing PROXY Protocol v1 in Python

```python
import socket
import ipaddress
from dataclasses import dataclass
from typing import Optional

@dataclass
class ProxyHeader:
    protocol: str       # TCP4, TCP6, UNKNOWN
    src_ip: str
    dst_ip: str
    src_port: int
    dst_port: int
    is_ipv6: bool

def parse_proxy_protocol_v1(data: bytes) -> Optional[ProxyHeader]:
    """
    Parse a PROXY Protocol v1 header from raw bytes.
    Returns None if not a valid PROXY Protocol header.
    """
    if not data.startswith(b'PROXY '):
        return None

    try:
        # Find the end of the header line
        crlf = data.find(b'\r\n')
        if crlf == -1:
            return None

        header_line = data[:crlf].decode('ascii')
        parts = header_line.split(' ')

        if len(parts) < 2:
            return None

        protocol = parts[1]

        if protocol == 'UNKNOWN':
            return ProxyHeader('UNKNOWN', '', '', 0, 0, False)

        if len(parts) != 6:
            return None

        _, proto, src_ip, dst_ip, src_port, dst_port = parts
        is_ipv6 = proto == 'TCP6'

        # Validate IPv6 addresses
        if is_ipv6:
            ipaddress.IPv6Address(src_ip)
            ipaddress.IPv6Address(dst_ip)

        return ProxyHeader(
            protocol=proto,
            src_ip=src_ip,
            dst_ip=dst_ip,
            src_port=int(src_port),
            dst_port=int(dst_port),
            is_ipv6=is_ipv6
        )
    except (ValueError, IndexError):
        return None

# Test
header_bytes = b'PROXY TCP6 2001:db8::1 2001:db8::10 54321 443\r\n'
result = parse_proxy_protocol_v1(header_bytes)
if result:
    print(f"Client: [{result.src_ip}]:{result.src_port}")
    print(f"Server: [{result.dst_ip}]:{result.dst_port}")
    print(f"Protocol: {result.protocol}, IPv6: {result.is_ipv6}")
```

## Server that Reads PROXY Protocol

```python
import socket

def handle_proxy_protocol_connection(conn: socket.socket):
    """Accept a connection using PROXY Protocol and read client IP."""
    # Read initial data (PROXY Protocol header + first chunk of app data)
    data = conn.recv(4096)

    # Parse the PROXY Protocol header
    header = parse_proxy_protocol_v1(data)
    if header:
        real_client_ip = header.src_ip
        # Strip the PROXY Protocol line from data
        body_start = data.find(b'\r\n') + 2
        app_data = data[body_start:]
    else:
        real_client_ip = conn.getpeername()[0]
        app_data = data

    print(f"Real client IP: {real_client_ip}")
    print(f"Application data: {app_data[:100]}")
    conn.close()
```

## Testing PROXY Protocol with IPv6

```bash
# Send a manual PROXY Protocol v1 header with IPv6
echo -e "PROXY TCP6 2001:db8::1 2001:db8::10 54321 80\r\nGET / HTTP/1.0\r\nHost: example.com\r\n\r\n" | \
    nc localhost 8080

# Use haproxy-debugger or nc to simulate
# Verify Nginx logs show the IPv6 client IP, not the HAProxy IP
sudo tail -f /var/log/nginx/access.log
```

## Conclusion

PROXY Protocol v1 encodes IPv6 addresses using the `TCP6` type in its text format. Configure HAProxy with `send-proxy-v2` to forward IPv6 client IPs to backends, and Nginx with `proxy_protocol` listen directive and `real_ip_header proxy_protocol` to extract them. Always restrict PROXY Protocol acceptance to trusted upstream proxy IP ranges.
