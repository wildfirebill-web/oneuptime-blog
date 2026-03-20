# How to Troubleshoot DNS Resolution Issues in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, DNS, Troubleshooting, Docker Networking, Container Networking

Description: Learn how to diagnose and fix DNS resolution issues in Docker containers managed by Portainer, including inter-container DNS and external name resolution.

---

DNS resolution failures are one of the most common networking issues in Docker. This guide covers diagnosing and fixing both inter-container DNS (service names) and external DNS (internet hostnames) problems.

## Diagnosing DNS Issues

Test DNS from inside a container:

```bash
# Test internal DNS (service name resolution)
docker exec -it $(docker ps -qf name=api) nslookup postgres

# Test external DNS
docker exec -it $(docker ps -qf name=api) nslookup google.com

# Check which DNS server the container is using
docker exec -it $(docker ps -qf name=api) cat /etc/resolv.conf

# Test with dig for more detail
docker exec -it $(docker ps -qf name=api) dig google.com +short
```

## Common DNS Issues and Fixes

### Issue 1: Service Name Not Resolving

Symptom: `nslookup postgres` returns `NXDOMAIN`.

Cause: Containers are on different networks, or one uses the default `docker0` bridge which doesn't support DNS.

```bash
# Check which network each container is on
docker inspect $(docker ps -qf name=api) | jq '.[0].NetworkSettings.Networks | keys'
docker inspect $(docker ps -qf name=postgres) | jq '.[0].NetworkSettings.Networks | keys'

# If they're on different networks, connect the api container to the database network
docker network connect my-app_backend $(docker ps -qf name=api)
```

### Issue 2: External DNS Not Working

Symptom: `nslookup google.com` times out.

Cause: The container's DNS server (usually `127.0.0.11`) cannot forward queries.

```bash
# Check if the host has working DNS
nslookup google.com

# Check Docker's embedded DNS resolver
docker exec -it $(docker ps -qf name=api) cat /etc/resolv.conf
# Should show: nameserver 127.0.0.11

# Test directly against a public DNS server
docker exec -it $(docker ps -qf name=api) nslookup google.com 8.8.8.8
```

Fix by specifying a custom DNS server for all containers in `/etc/docker/daemon.json`:

```json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
```

Then restart Docker: `sudo systemctl restart docker`

### Issue 3: Intermittent DNS Failures

Symptom: DNS works sometimes but fails randomly.

Cause: DNS timeout under high load or `ndots` setting causing unnecessary external lookups.

```yaml
# Fix in your Compose stack by tuning DNS options
services:
  api:
    dns:
      - 127.0.0.11
      - 8.8.8.8
    dns_search:
      - .
    dns_opt:
      - ndots:1      # Reduce from default 5 — fewer unnecessary lookups
      - timeout:2    # Seconds before retry
      - attempts:3   # Retry count
```

### Issue 4: Slow DNS Lookups

Symptom: Application startup is slow; DNS queries take >1 second.

Cause: High `ndots` value causing the resolver to try multiple search domains before the bare hostname.

```bash
# Check the current ndots setting
docker exec -it $(docker ps -qf name=api) cat /etc/resolv.conf | grep options

# Set ndots:1 in your stack (see above)
```

### Issue 5: DNS on the Default Bridge Network

Containers on the default `docker0` bridge cannot use DNS names — only IPs. Move containers to a custom network:

```yaml
services:
  api:
    networks:
      - custom_net   # Custom bridge supports DNS

networks:
  custom_net:
    driver: bridge
```

## Network-Level DNS Configuration

Override DNS per-network in a Compose stack:

```yaml
networks:
  app_net:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: my-bridge0
```

## Checking Docker's Embedded DNS

Docker's embedded DNS resolver runs at `127.0.0.11:53` inside each container. If this is unreachable:

```bash
# Test DNS directly
docker exec -it $(docker ps -qf name=api) \
  nslookup postgres 127.0.0.11

# Check iptables rules for the Docker embedded DNS
sudo iptables -L DOCKER -n | grep 53
```
