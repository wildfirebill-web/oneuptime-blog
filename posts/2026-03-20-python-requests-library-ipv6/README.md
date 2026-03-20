# How to Use Python requests Library with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv6, requests, HTTP, REST API, Networking

Description: Use Python's requests library to make HTTP requests to IPv6 endpoints, handle IPv6 URLs, and force IPv6 connections.

## IPv6 URLs in Requests

IPv6 addresses in URLs must be enclosed in square brackets (RFC 2732):

```python
import requests

# HTTP request to an IPv6 server
# IPv6 address in URL requires square brackets
response = requests.get("http://[2001:db8::1]/api/status")
print(response.status_code)
print(response.json())

# HTTPS to IPv6 endpoint
response = requests.get(
    "https://[2001:db8::1]:8443/api",
    verify=False  # Skip cert check for self-signed certs in testing
)
```

## Forcing IPv6 for Hostname Connections

By default, `requests` uses whatever address Python resolves (may be IPv4 or IPv6). Force IPv6 using a custom transport adapter:

```python
import requests
import socket
from requests.adapters import HTTPAdapter
from urllib3.connection import HTTPConnection, HTTPSConnection
from urllib3.connectionpool import HTTPConnectionPool, HTTPSConnectionPool

class IPv6Connection(HTTPConnection):
    """HTTP connection that forces IPv6."""
    def connect(self):
        # Resolve hostname to IPv6 specifically
        addrs = socket.getaddrinfo(
            self.host, self.port,
            family=socket.AF_INET6,
            type=socket.SOCK_STREAM
        )
        if not addrs:
            raise ConnectionError(f"No IPv6 address for {self.host}")
        af, socktype, proto, canonname, sockaddr = addrs[0]
        self.sock = socket.socket(af, socktype)
        self.sock.connect(sockaddr)

class IPv6ConnectionPool(HTTPConnectionPool):
    ConnectionCls = IPv6Connection

class IPv6Adapter(HTTPAdapter):
    def get_connection(self, url, proxies=None):
        return IPv6ConnectionPool(self._get_host(url), self._get_port(url))

# Mount the IPv6 adapter for specific hosts
session = requests.Session()
session.mount("http://ipv6.example.com", IPv6Adapter())

response = session.get("http://ipv6.example.com/api")
```

## Simpler Approach: Force IPv6 via getaddrinfo Patch

A simpler approach for testing: monkey-patch socket.getaddrinfo to prefer IPv6:

```python
import requests
import socket

# Save original getaddrinfo
original_getaddrinfo = socket.getaddrinfo

def ipv6_getaddrinfo(host, port, family=0, type=0, proto=0, flags=0):
    """Prefer IPv6 addresses in resolution."""
    results = original_getaddrinfo(host, port, family, type, proto, flags)
    # Sort IPv6 first
    ipv6_results = [r for r in results if r[0] == socket.AF_INET6]
    ipv4_results = [r for r in results if r[0] == socket.AF_INET]
    return ipv6_results + ipv4_results

# Apply patch
socket.getaddrinfo = ipv6_getaddrinfo

# Now requests will prefer IPv6 when both are available
response = requests.get("https://google.com")
print(f"Connected via: {'IPv6' if ':' in response.raw._connection.sock.getpeername()[0] else 'IPv4'}")

# Restore original
socket.getaddrinfo = original_getaddrinfo
```

## Testing IPv6 Connectivity with requests

```python
import requests

def test_ipv6_connectivity() -> dict:
    """Test IPv6 connectivity using external services."""
    results = {}

    # Test 1: Direct IPv6 address
    try:
        r = requests.get("http://[2001:4860:4860::8888]", timeout=5)
        results["direct_ipv6"] = "reachable"
    except requests.exceptions.ConnectionError:
        results["direct_ipv6"] = "unreachable"

    # Test 2: IPv6 test site
    try:
        r = requests.get("https://ipv6.icanhazip.com", timeout=5)
        if ':' in r.text.strip():
            results["ipv6_internet"] = r.text.strip()
        else:
            results["ipv6_internet"] = "connected via IPv4 despite request"
    except requests.exceptions.ConnectionError:
        results["ipv6_internet"] = "unreachable"

    return results

info = test_ipv6_connectivity()
for k, v in info.items():
    print(f"{k}: {v}")
```

## Making API Calls to IPv6 Services

When interacting with APIs hosted on IPv6-only or IPv6-preferred endpoints:

```python
import requests

class IPv6APIClient:
    """API client with IPv6 endpoint support."""

    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "Accept": "application/json",
            "Content-Type": "application/json"
        })

    def get(self, path: str, **kwargs):
        """Make a GET request to the IPv6 API."""
        url = f"{self.base_url}{path}"
        return self.session.get(url, **kwargs)

    def post(self, path: str, data: dict, **kwargs):
        """Make a POST request to the IPv6 API."""
        url = f"{self.base_url}{path}"
        return self.session.post(url, json=data, **kwargs)

# Example: API hosted on IPv6
client = IPv6APIClient("http://[2001:db8::api]/v1")
# response = client.get("/devices")
# response = client.post("/devices", {"name": "router-1", "address": "2001:db8::1"})
```

## Conclusion

Python's `requests` library works with IPv6 when URLs use the bracket notation for IPv6 addresses. To force IPv6 for hostname connections, use a custom adapter or monkey-patch `socket.getaddrinfo`. For production code, consider using `httpx` which has better built-in IPv6 support and async capabilities.
