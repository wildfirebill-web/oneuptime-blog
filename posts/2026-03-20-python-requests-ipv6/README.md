# How to Make IPv6 HTTP Requests with Python requests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Python, requests, HTTP, urllib3

Description: Make HTTP requests to IPv6-only servers and dual-stack hosts using Python's requests library, handling IPv6 URL formatting, source address binding, and Happy Eyeballs connection preferences.

## Basic IPv6 HTTP Requests

```python
import requests

# HTTP request to IPv6 address — brackets required in URL
response = requests.get("http://[2001:db8::1]/")
print(response.status_code)

# HTTPS to IPv6 address
response = requests.get("https://[2001:db8::1]/api/v1/status")
print(response.json())

# IPv6-enabled domain (requests resolves AAAA automatically)
response = requests.get("https://ipv6.google.com")
print(response.status_code)

# Check which IP was used
import socket
session = requests.Session()
adapter = requests.adapters.HTTPAdapter()
session.mount("https://", adapter)

resp = session.get("https://ifconfig.co/json")
data = resp.json()
print(f"Connected from: {data.get('ip')}")
```

## Force IPv6-Only Connections

```python
import requests
import socket
import urllib3

class IPv6Adapter(requests.adapters.HTTPAdapter):
    """HTTP adapter that forces IPv6-only connections."""

    def send(self, request, *args, **kwargs):
        # Override DNS resolution to force IPv6
        old_getaddrinfo = socket.getaddrinfo

        def ipv6_getaddrinfo(host, port, family=0, type=0, proto=0, flags=0):
            return old_getaddrinfo(host, port,
                                   socket.AF_INET6,  # force IPv6
                                   type, proto, flags)

        socket.getaddrinfo = ipv6_getaddrinfo
        try:
            return super().send(request, *args, **kwargs)
        finally:
            socket.getaddrinfo = old_getaddrinfo

# Use the IPv6-only adapter
session = requests.Session()
session.mount("https://", IPv6Adapter())
session.mount("http://", IPv6Adapter())

try:
    resp = session.get("https://google.com", timeout=10)
    print(f"Status: {resp.status_code}")
except requests.exceptions.ConnectionError as e:
    print(f"IPv6 not available: {e}")
```

## Bind to Specific IPv6 Source Address

```python
import requests
import socket

class SourceBoundAdapter(requests.adapters.HTTPAdapter):
    """Bind outgoing connections to a specific local IPv6 address."""

    def __init__(self, source_address: str, *args, **kwargs):
        self.source_address = source_address
        super().__init__(*args, **kwargs)

    def init_poolmanager(self, *args, **kwargs):
        kwargs["socket_options"] = [
            (socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 1),
        ]
        # urllib3 source_address is (host, port) tuple
        kwargs["source_address"] = (self.source_address, 0)
        super().init_poolmanager(*args, **kwargs)

# Bind requests to a specific IPv6 address
session = requests.Session()
adapter = SourceBoundAdapter("2001:db8::100")
session.mount("https://", adapter)
session.mount("http://", adapter)

resp = session.get("https://ifconfig.co")
print(f"Public IP: {resp.text.strip()}")  # Should show 2001:db8::100
```

## Handle Dual-Stack with Timeout

```python
import requests
import concurrent.futures

def fetch_dual_stack(url: str, timeout: float = 5.0) -> requests.Response:
    """
    Fetch URL preferring IPv6, fall back to IPv4.
    Implements a simplified Happy Eyeballs approach.
    """
    ipv6_url = url
    ipv4_url = url

    def try_ipv6():
        s = requests.Session()
        s.mount("https://", IPv6Adapter())
        s.mount("http://", IPv6Adapter())
        return s.get(ipv6_url, timeout=timeout)

    def try_ipv4():
        import time
        time.sleep(0.05)   # 50ms delay for IPv4 (Happy Eyeballs)
        return requests.get(ipv4_url, timeout=timeout)

    with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
        futures = {
            executor.submit(try_ipv6): "ipv6",
            executor.submit(try_ipv4): "ipv4",
        }
        for future in concurrent.futures.as_completed(futures):
            transport = futures[future]
            try:
                result = future.result()
                print(f"Connected via {transport}")
                return result
            except Exception:
                continue

    raise requests.exceptions.ConnectionError("Both IPv4 and IPv6 failed")

# Usage
resp = fetch_dual_stack("https://google.com")
print(f"Status: {resp.status_code}")
```

## IPv6 HTTPS with Custom CA

```python
import requests

# HTTPS to IPv6 address with custom CA bundle
response = requests.get(
    "https://[2001:db8::1]/api/v1/health",
    verify="/etc/ssl/certs/my-ca-bundle.crt",
    timeout=10,
)

# Or disable verification for testing (never in production)
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
response = requests.get("https://[2001:db8::1]/", verify=False, timeout=5)
```

## Proxy Over IPv6

```python
import requests

# Use an IPv6 SOCKS5 proxy
proxies = {
    "http":  "socks5h://[2001:db8::proxy]:1080",
    "https": "socks5h://[2001:db8::proxy]:1080",
}

# pip install requests[socks]
resp = requests.get("https://example.com", proxies=proxies, timeout=10)
print(resp.status_code)

# HTTP proxy over IPv6
proxies_http = {
    "http":  "http://[2001:db8::proxy]:3128",
    "https": "http://[2001:db8::proxy]:3128",
}
resp = requests.get("https://example.com", proxies=proxies_http)
```

## Conclusion

Python's `requests` library handles IPv6 addresses in URLs with bracket notation (`http://[2001:db8::1]/`). For domain names, `requests` automatically uses AAAA records when available; force IPv6-only by overriding `socket.getaddrinfo` with `AF_INET6` in a custom `HTTPAdapter`. To bind to a specific source IPv6 address, set `source_address` in `urllib3.PoolManager` via a custom adapter. For dual-stack applications, implement a simplified Happy Eyeballs approach by racing IPv6 and delayed IPv4 connections in parallel threads. Use `socks5h://[IPv6-addr]:port` in the proxies dict to route through an IPv6 SOCKS proxy, which also handles DNS resolution remotely.
