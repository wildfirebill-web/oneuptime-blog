# How to Enable IPv6 in Docker Daemon Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Daemon, Configuration, Networking

Description: Enable IPv6 support in the Docker daemon by editing daemon.json, configure IPv6 address pools, and restart Docker to provide IPv6 connectivity to containers.

## Introduction

Docker does not enable IPv6 by default. To use IPv6 in containers, you must enable it in the Docker daemon configuration file `/etc/docker/daemon.json`. Once enabled, Docker assigns IPv6 addresses to containers from the configured ULA prefix. The daemon must be restarted after configuration changes for them to take effect.

## Enable IPv6 in daemon.json

```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/48",
  "ip6tables": true,
  "experimental": false
}
```

```bash
# Apply configuration
sudo systemctl restart docker

# Verify IPv6 is enabled
docker info | grep -i ipv6
# Output: IPv6: true

# Check daemon configuration was applied
docker network inspect bridge | grep -A5 "EnableIPv6"
```

## Full daemon.json with IPv6 and Logging

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/48",
  "ip6tables": true,
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    },
    {
      "base": "fd00::/48",
      "size": 64
    }
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["8.8.8.8", "2001:4860:4860::8888"]
}
```

## Verify Configuration

```bash
# Check Docker is listening on both IPv4 and IPv6
ss -tlnp | grep dockerd

# Inspect default bridge network for IPv6
docker network inspect bridge | python3 -c "
import json, sys
data = json.load(sys.stdin)
print('EnableIPv6:', data[0]['EnableIPv6'])
print('IPv6Subnet:', data[0]['IPAM']['Config'])
"

# Create a container and check IPv6
docker run --rm alpine ip -6 addr show eth0
```

## Troubleshooting daemon.json Changes

```bash
# Check for JSON syntax errors
cat /etc/docker/daemon.json | python3 -m json.tool

# View daemon logs if restart fails
journalctl -u docker.service -n 50

# Validate daemon is running
systemctl status docker

# Test IPv6 connectivity from a container
docker run --rm alpine ping6 -c 3 2001:4860:4860::8888
```

## Conclusion

Enable Docker IPv6 by setting `"ipv6": true` and `"fixed-cidr-v6"` in `/etc/docker/daemon.json`, then restart the Docker daemon with `systemctl restart docker`. The `ip6tables` flag enables network isolation rules for IPv6. Use a ULA prefix (`fd00::/8`) for container networks unless you have a routable IPv6 prefix available. Verify with `docker info | grep IPv6` and test with `docker run --rm alpine ping6 2001:4860:4860::8888`.
