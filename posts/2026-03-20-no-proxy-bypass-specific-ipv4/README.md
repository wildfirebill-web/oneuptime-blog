# How to Bypass Proxy for Specific IPv4 Addresses Using no_proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Proxy, No_proxy, IPv4, Bypass, Environment Variables

Description: Configure no_proxy and NO_PROXY environment variables to exclude specific IPv4 addresses, CIDR ranges, and domain suffixes from proxy routing on Linux.

## Introduction

When using an HTTP proxy, you typically want internal services - databases, internal APIs, local services, cloud metadata endpoints - to connect directly without going through the proxy. The `no_proxy` environment variable lists addresses that bypass the proxy. Different tools parse `no_proxy` differently, so understanding the syntax and tool-specific behavior is important.

## Basic no_proxy Syntax

```bash
# Comma-separated list of hosts, IPs, CIDR ranges, and domain suffixes

export no_proxy="localhost,127.0.0.1,::1"

# Add specific IPv4 addresses
export no_proxy="localhost,127.0.0.1,10.0.1.10,10.0.1.11"

# Add subnets (CIDR notation - supported by most tools)
export no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"

# Add domain suffixes (leading dot = wildcard subdomain)
export no_proxy="localhost,127.0.0.1,.corp.example.com,.internal"
```

## Comprehensive no_proxy Configuration

```bash
export no_proxy="\
localhost,\
127.0.0.1,\
127.0.0.0/8,\
::1,\
10.0.0.0/8,\
172.16.0.0/12,\
192.168.0.0/16,\
.local,\
.corp.example.com,\
.svc.cluster.local,\
.cluster.local,\
169.254.169.254,\
metadata.google.internal"

# UPPERCASE variant (required by some tools)
export NO_PROXY="$no_proxy"
```

Note: `169.254.169.254` is the cloud metadata endpoint (AWS, GCP, Azure) - always bypass the proxy for it.

## Per-Tool no_proxy Behavior

### curl

curl supports CIDR notation in `no_proxy` since version 7.86:

```bash
# Test: curl should bypass proxy for 10.0.1.10
export http_proxy="http://proxy:3128"
export no_proxy="10.0.0.0/8"
curl http://10.0.1.10/api  # Direct connection
curl http://8.8.8.8        # Goes through proxy
```

### wget

wget does NOT support CIDR in no_proxy - use individual IPs or domain patterns:

```bash
# ~/.wgetrc
no_proxy = localhost,127.0.0.1,10.0.1.10,10.0.1.11,.corp.example.com
```

### Python requests

```python
import requests

proxies = {
    'http': 'http://proxy.corp.example.com:3128',
    'https': 'http://proxy.corp.example.com:3128',
}

# Pass proxies dict; no_proxy behavior comes from NO_PROXY env var
# Or explicitly bypass:
session = requests.Session()
session.proxies = {'http': 'http://proxy:3128', 'https': 'http://proxy:3128'}

# For a specific request, bypass proxy:
response = requests.get('http://10.0.1.10/api', proxies={'http': None, 'https': None})
```

### git

```bash
# Bypass proxy for internal git server
git config --global http.proxy "http://proxy:3128"
git config --global http.https://gitlab.corp.example.com.proxy ""
# Empty string means no proxy for that host

# Or use no_proxy env var
export no_proxy=".corp.example.com"
git clone https://gitlab.corp.example.com/my-repo.git
```

### Docker (pulling images)

```bash
# Docker daemon no_proxy (in daemon.json or systemd override)
sudo tee /etc/systemd/system/docker.service.d/proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=http://proxy.corp.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,myregistry.internal,.corp.example.com"
EOF
```

## Testing no_proxy Behavior

```bash
# Verify curl bypasses proxy for an internal address
curl -v http://10.0.1.10/api 2>&1 | grep -E "(Trying|proxy|direct)"

# Should show "Trying 10.0.1.10..." without mentioning the proxy

# Compare with an external address (should route via proxy)
curl -v http://example.com 2>&1 | grep "Trying"
# Should show "Trying proxy.corp.example.com..."
```

## AWS/GCP/Azure Metadata Endpoint Bypass

```bash
# Always bypass proxy for cloud metadata services
export no_proxy="169.254.169.254,metadata.google.internal,169.254.169.254/32"

# Test direct access to AWS EC2 metadata
curl http://169.254.169.254/latest/meta-data/local-ipv4
```

## Kubernetes in-Cluster Bypass

```bash
# Kubernetes DNS and service CIDRs should bypass proxy
export no_proxy="\
10.96.0.0/12,\
10.244.0.0/16,\
.svc.cluster.local,\
.cluster.local,\
.default.svc"
```

## no_proxy with Wildcards

`*` as a wildcard (entire bypasses proxy for everything):

```bash
# Disable proxy entirely
export no_proxy="*"

# Bypass all .example.com subdomains (use leading dot, not *)
export no_proxy=".example.com"
# This matches api.example.com, app.example.com, etc.
```

## Conclusion

Configure `no_proxy` with all RFC 1918 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), cloud metadata endpoints (`169.254.169.254`), and internal domain suffixes. Always set both `no_proxy` and `NO_PROXY` (uppercase) for maximum compatibility. CIDR notation works in curl and many modern tools; wget requires individual IPs or domain patterns. Test with `curl -v` to confirm which connections bypass the proxy.
