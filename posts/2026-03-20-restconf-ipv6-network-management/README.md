# How to Use RESTCONF for IPv6 Network Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RESTCONF, IPv6, Network Automation, REST API, YANG, Python, RFC 8040

Description: Use RESTCONF (RFC 8040) with Python to manage IPv6 network configurations via REST API, including GET, PUT, PATCH, and DELETE operations on YANG data models.

## Introduction

RESTCONF (RFC 8040) exposes YANG-modeled network configuration as a REST API over HTTPS. It is easier to use than NETCONF for developers already familiar with REST APIs. RESTCONF supports the same YANG data models as NETCONF and is available on Cisco IOS-XE 16.3+, IOS-XR 6.1+, and Juniper Junos 17.3+.

## Step 1: RESTCONF Client Setup

```python
# restconf/client.py

import requests
import json
from urllib3.exceptions import InsecureRequestWarning
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

class RESTCONFClient:
    """RESTCONF client for IPv6-capable network devices."""

    def __init__(self, host: str, username: str, password: str,
                 port: int = 443):
        # host can be IPv6 address: 2001:db8::r1
        if ":" in host and not host.startswith("["):
            host = f"[{host}]"  # Bracket notation for URL
        self.base_url = f"https://{host}:{port}/restconf"
        self.auth = (username, password)
        self.headers = {
            "Accept": "application/yang-data+json",
            "Content-Type": "application/yang-data+json",
        }

    def get(self, path: str) -> dict:
        url = f"{self.base_url}/{path}"
        r = requests.get(url, auth=self.auth, headers=self.headers, verify=False)
        r.raise_for_status()
        return r.json()

    def put(self, path: str, data: dict) -> requests.Response:
        url = f"{self.base_url}/{path}"
        r = requests.put(url, auth=self.auth, headers=self.headers,
                        json=data, verify=False)
        r.raise_for_status()
        return r

    def patch(self, path: str, data: dict) -> requests.Response:
        url = f"{self.base_url}/{path}"
        r = requests.patch(url, auth=self.auth, headers=self.headers,
                          json=data, verify=False)
        r.raise_for_status()
        return r

    def delete(self, path: str) -> requests.Response:
        url = f"{self.base_url}/{path}"
        r = requests.delete(url, auth=self.auth, headers=self.headers, verify=False)
        r.raise_for_status()
        return r
```

## Step 2: Get IPv6 Interface Configuration

```python
# restconf/get_ipv6.py
from client import RESTCONFClient

def get_ipv6_interfaces(client: RESTCONFClient) -> list:
    """Get all IPv6 addresses via RESTCONF IETF interfaces model."""
    path = "data/ietf-interfaces:interfaces"
    data = client.get(path)

    ipv6_addresses = []
    for iface in data.get("ietf-interfaces:interfaces", {}).get("interface", []):
        name = iface.get("name")
        ipv6 = iface.get("ietf-ip:ipv6", {})
        for addr in ipv6.get("address", []):
            ipv6_addresses.append({
                "interface": name,
                "address": addr.get("ip"),
                "prefix_length": addr.get("prefix-length"),
            })

    return ipv6_addresses

client = RESTCONFClient("2001:db8::r1", "admin", "secret")
addrs = get_ipv6_interfaces(client)
for a in addrs:
    print(f"{a['interface']}: {a['address']}/{a['prefix_length']}")
```

## Step 3: Configure IPv6 Address via PATCH

```python
def configure_ipv6_address(client: RESTCONFClient, interface: str,
                             ipv6_addr: str, prefix_len: int):
    """Add an IPv6 address using RESTCONF PATCH."""
    path = f"data/ietf-interfaces:interfaces/interface={interface}"

    payload = {
        "ietf-interfaces:interface": {
            "name": interface,
            "ietf-ip:ipv6": {
                "enabled": True,
                "address": [
                    {
                        "ip": ipv6_addr,
                        "prefix-length": prefix_len,
                    }
                ]
            }
        }
    }

    response = client.patch(path, payload)
    print(f"Status: {response.status_code}")
    return response.status_code == 204  # 204 No Content = success
```

## Step 4: Delete IPv6 Address

```python
def delete_ipv6_address(client: RESTCONFClient, interface: str, ipv6_addr: str):
    """Remove an IPv6 address using RESTCONF DELETE."""
    path = (
        f"data/ietf-interfaces:interfaces/interface={interface}"
        f"/ietf-ip:ipv6/address={ipv6_addr}"
    )
    response = client.delete(path)
    return response.status_code == 204
```

## Step 5: Get IPv6 Routes via RESTCONF

```python
def get_ipv6_routes(client: RESTCONFClient) -> list:
    """Retrieve IPv6 routing table via RESTCONF."""
    path = "data/ietf-routing:routing/ribs/rib=ipv6-unicast/routes"
    try:
        data = client.get(path)
        return data.get("ietf-routing:routes", {}).get("route", [])
    except Exception as e:
        print(f"Error retrieving IPv6 routes: {e}")
        return []
```

## Step 6: Test RESTCONF Connectivity

```bash
# Test RESTCONF root
curl -k -u admin:secret \
    -H "Accept: application/yang-data+json" \
    "https://[2001:db8::r1]:443/restconf/"

# Get all interfaces
curl -k -u admin:secret \
    -H "Accept: application/yang-data+json" \
    "https://[2001:db8::r1]:443/restconf/data/ietf-interfaces:interfaces" \
    | python3 -m json.tool

# PATCH IPv6 address
curl -k -u admin:secret \
    -X PATCH \
    -H "Content-Type: application/yang-data+json" \
    -H "Accept: application/yang-data+json" \
    -d '{"ietf-interfaces:interface":{"name":"GigabitEthernet0/0","ietf-ip:ipv6":{"address":[{"ip":"2001:db8::1","prefix-length":64}]}}}' \
    "https://[2001:db8::r1]:443/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet0/0"
```

## Conclusion

RESTCONF provides a developer-friendly REST API for IPv6 network configuration, backed by YANG data models. Use bracket notation for IPv6 addresses in URLs: `https://[2001:db8::1]:443/restconf/`. PATCH is idempotent and suitable for adding addresses without replacing existing ones. Monitor RESTCONF API health and response times with OneUptime's HTTPS checks on management plane endpoints.
