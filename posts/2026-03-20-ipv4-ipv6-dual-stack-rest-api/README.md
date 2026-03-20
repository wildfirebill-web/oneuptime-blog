# How to Handle IPv4 and IPv6 Dual-Stack in REST API Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: REST API, IPv4, IPv6, Dual-Stack, Python, Node.js, Networking

Description: Learn how to configure REST API servers to handle both IPv4 and IPv6 dual-stack connections, normalize addresses, and build IP classification logic that works across both protocols.

## Understanding Dual-Stack

On dual-stack systems, binding to `::` (IPv6 any) or `0.0.0.0` (IPv4 any) has different behavior depending on the OS and `IPV6_V6ONLY` socket option.

| Bind Address | Accepts IPv4? | Accepts IPv6? |
|---|---|---|
| `0.0.0.0` | Yes | No |
| `127.0.0.1` | Loopback IPv4 only | No |
| `::` (Linux default) | Yes (mapped) | Yes |
| `::1` | No | Loopback IPv6 only |

## Python: Bind to Both Protocols

```python
import socket
import threading
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        client_ip = self.client_address[0]
        # Normalise IPv4-mapped IPv6 (::ffff:192.168.1.1) to plain IPv4
        if client_ip.startswith("::ffff:"):
            client_ip = client_ip[7:]
        self.send_response(200)
        self.end_headers()
        self.wfile.write(f"Your IP: {client_ip}".encode())

def start_server(host: str, port: int, family: int) -> HTTPServer:
    server = HTTPServer.__new__(HTTPServer)
    server.socket = socket.socket(family, socket.SOCK_STREAM)
    server.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    if family == socket.AF_INET6:
        server.socket.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 1)
    server.socket.bind((host, port))
    server.server_address = (host, port)
    server.RequestHandlerClass = Handler
    return server

ipv4_server = start_server("0.0.0.0", 8080, socket.AF_INET)
ipv6_server = start_server("::",     8080, socket.AF_INET6)

threading.Thread(target=ipv4_server.serve_forever, daemon=True).start()
ipv6_server.serve_forever()
```

## Node.js: Express Dual-Stack

```javascript
const express = require("express");
const app = express();

// Express/Node.js automatically accepts both IPv4 and IPv6 when listening on "::"
// Node uses IPV6_V6ONLY=0 by default on Linux (dual-stack)

app.get("/whoami", (req, res) => {
    let ip = req.socket.remoteAddress;
    // Normalise ::ffff:x.x.x.x (IPv4-mapped IPv6) to plain IPv4
    if (ip.startsWith("::ffff:")) {
        ip = ip.slice(7);
    }
    res.json({ ip, family: ip.includes(":") ? "ipv6" : "ipv4" });
});

// "::" on Linux accepts both IPv4 (mapped) and IPv6 connections
app.listen(8080, "::", () => console.log("Listening on dual-stack :8080"));
```

## Normalizing IPv4-Mapped IPv6 Addresses

```python
import ipaddress

def normalize_client_ip(raw: str) -> str:
    """
    Convert IPv4-mapped IPv6 addresses to plain IPv4.
    ::ffff:192.168.1.1 → 192.168.1.1
    """
    if raw.startswith("::ffff:"):
        raw = raw[7:]
    try:
        addr = ipaddress.ip_address(raw)
        if isinstance(addr, ipaddress.IPv6Address) and addr.ipv4_mapped:
            return str(addr.ipv4_mapped)
        return str(addr)
    except ValueError:
        return raw

print(normalize_client_ip("::ffff:192.168.1.1"))  # 192.168.1.1
print(normalize_client_ip("::1"))                 # ::1
print(normalize_client_ip("192.168.1.1"))         # 192.168.1.1
```

## Classifying IP Version

```python
import ipaddress

def ip_version(s: str) -> str:
    s = normalize_client_ip(s)
    try:
        addr = ipaddress.ip_address(s)
        return "ipv4" if isinstance(addr, ipaddress.IPv4Address) else "ipv6"
    except ValueError:
        return "invalid"
```

## Conclusion

Binding to `::` with `IPV6_V6ONLY=0` (Linux default) accepts both IPv4 and IPv6 connections, with IPv4 clients appearing as `::ffff:x.x.x.x`. Always normalize these mapped addresses before IP-based logic (whitelisting, geolocation, logging). If you need strict separation, bind two sockets: one to `0.0.0.0` and one to `::` with `IPV6_V6ONLY=1`. On macOS/BSD, `IPV6_V6ONLY=1` is the default, so you always need two sockets for dual-stack.
