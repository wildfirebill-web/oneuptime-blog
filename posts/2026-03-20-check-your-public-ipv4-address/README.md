# How to Check Your Public IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Networking, Public IP, NAT, Network Diagnostics

Description: Your public IPv4 address is the address visible to the internet from behind your NAT router, and it can be discovered using web services, DNS queries, or command-line tools.

## Why Your Local IP Is Not Your Public IP

If your machine has `192.168.1.100`, that is your private RFC 1918 address. Your ISP-assigned public IP is on your router's WAN interface. Any host on the internet sees your router's public IP, not your device's private IP.

## Method 1: curl with IP Discovery Services

```bash
# Simple — returns just the IP address
curl -s https://api.ipify.org
curl -s https://ifconfig.me
curl -s https://icanhazip.com
curl -s https://ipecho.net/plain

# JSON response with additional info
curl -s https://ipinfo.io
curl -s https://api.ipify.org?format=json
```

## Method 2: DNS-Based Lookup (No HTTP Required)

Some DNS servers have a special hostname that returns your public IP:

```bash
# Using OpenDNS resolver — returns your public IP
dig +short myip.opendns.com @resolver1.opendns.com

# Google's TXT record approach
dig +short o-o.myaddr.l.google.com @ns1.google.com TXT
```

## Method 3: Python Script

```python
import urllib.request
import json

def get_public_ip() -> str:
    """Retrieve public IP using ipify API."""
    url = "https://api.ipify.org?format=json"
    with urllib.request.urlopen(url, timeout=5) as response:
        data = json.loads(response.read().decode())
        return data["ip"]

def get_public_ip_info() -> dict:
    """Retrieve public IP and geolocation info from ipinfo.io."""
    url = "https://ipinfo.io/json"
    with urllib.request.urlopen(url, timeout=5) as response:
        return json.loads(response.read().decode())

ip = get_public_ip()
print(f"Public IP: {ip}")

info = get_public_ip_info()
print(f"City: {info.get('city')}, Country: {info.get('country')}")
print(f"ISP: {info.get('org')}")
```

## Method 4: Check the Router's WAN Interface

For server environments, check the WAN IP directly:

```bash
# If you have access to the router/gateway CLI
# Cisco IOS:
# show ip interface brief

# Linux router — check the WAN interface (e.g., ppp0 for PPPoE, eth0 for direct)
ip addr show ppp0
ip addr show eth0  # Look for the non-RFC1918 address

# Cloud instance: show all IPs
ip addr show
# The non-RFC1918 address is the public IP (or mapped via elastic IP)
```

## Monitoring Public IP Changes (DDNS Use Case)

```python
import time, urllib.request

def watch_ip_changes(interval_seconds: int = 60):
    """Monitor for public IP changes (useful for DDNS updates)."""
    last_ip = None
    while True:
        with urllib.request.urlopen("https://api.ipify.org", timeout=5) as r:
            current_ip = r.read().decode()
        if current_ip != last_ip:
            print(f"IP changed: {last_ip} -> {current_ip}")
            last_ip = current_ip
        time.sleep(interval_seconds)
```

## Key Takeaways

- Your private IP (192.168.x.x, etc.) is not your public internet IP.
- Use `curl https://api.ipify.org` or `dig myip.opendns.com @resolver1.opendns.com` for quick lookups.
- DNS-based methods work without HTTP and are useful in restricted environments.
- Public IPs change on home connections; use DDNS if you need a stable hostname.
