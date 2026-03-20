# How to Configure Docker to Use a Specific IPv4 Address Range

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, IPv4, daemon.json, Address Range, Configuration

Description: Configure Docker to allocate container IPs from a specific IPv4 address range using daemon.json default address pools and bip settings to avoid conflicts with your network infrastructure.

## Introduction

Docker by default selects subnets from `172.17.0.0/16` (default bridge) and `172.18.0.0/16`–`172.31.0.0/16` (custom networks). In environments where these overlap with VPN or corporate ranges, you need to redirect Docker to a safe, non-conflicting address range.

## Configure /etc/docker/daemon.json

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "bip": "10.200.0.1/24",
  "default-address-pools": [
    {
      "base": "10.200.0.0/16",
      "size": 24
    }
  ]
}
```

- `bip`: defines the IP and subnet for the default `docker0` bridge
- `default-address-pools`: defines the parent range and subnet size for new networks

```bash
sudo systemctl restart docker
```

## Verifying the Configuration

```bash
# Check docker0 bridge uses the new range

ip addr show docker0 | grep inet

# Create a test network and confirm it gets an address from the pool
docker network create test-range
docker network inspect test-range --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
# Should show 10.200.1.0/24 (or the next available /24 in the pool)

# Cleanup
docker network rm test-range
```

## Multiple Address Pools for Different Uses

```json
{
  "bip": "10.200.0.1/24",
  "default-address-pools": [
    {
      "base": "10.200.0.0/16",
      "size": 24
    },
    {
      "base": "10.201.0.0/16",
      "size": 28
    }
  ]
}
```

Docker uses the first pool until it is exhausted, then moves to the second.

## Choosing Safe Non-Conflicting Ranges

Run this script to find what ranges are in use on the host:

```bash
#!/bin/bash
echo "Currently in-use network ranges:"
ip route show | awk '{print $1}' | grep -v "^default\|^via\|^dev\|^cache"
```

Pick a range not in the output. Common safe choices:
- `10.200.0.0/16` (usually unused in enterprise)
- `100.64.0.0/10` (IANA CGNAT range, typically unused)
- `192.168.200.0/24` (if your LAN uses 192.168.1.x)

## Checking for Conflicts Before Applying

```bash
# Check if your chosen subnet conflicts with any existing route
ip route show | grep "10.200.0.0"
# No output = no conflict
```

## Applying to Docker Compose Projects

Custom networks defined without explicit subnets in Docker Compose will now use the configured address pools:

```yaml
networks:
  app-net:
    # No subnet specified - will use 10.200.x.0/24 from the pool
    driver: bridge
```

## Conclusion

Configure `bip` and `default-address-pools` in `/etc/docker/daemon.json` to redirect Docker to a non-conflicting IPv4 range. This is especially important before joining a VPN on a development machine or deploying to a cloud environment with existing RFC 1918 addressing.
