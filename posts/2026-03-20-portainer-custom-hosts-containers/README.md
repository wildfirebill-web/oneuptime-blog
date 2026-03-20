# How to Configure Custom Host File Entries for Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, /etc/hosts, Custom DNS, Networking, Container Configuration

Description: Learn how to add custom host file entries to Docker containers in Portainer using extra_hosts in Compose stacks or the Portainer container configuration UI.

---

Custom `/etc/hosts` entries let you map hostnames to specific IP addresses inside a container, overriding DNS. This is useful for pointing service names at specific IPs, testing against local environments, or working around DNS in air-gapped setups.

## Adding Extra Hosts in a Stack

Use the `extra_hosts` key in your Compose file:

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    extra_hosts:
      - "legacy-db:192.168.1.50"       # Internal server without DNS
      - "payment-gateway:10.0.0.200"   # Internal service
      - "host.docker.internal:host-gateway"  # Access the Docker host
    networks:
      - app_net

networks:
  app_net:
```

Verify the entries are injected:

```bash
docker exec -it $(docker ps -qf name=api) cat /etc/hosts
# Should show custom entries at the bottom
```

## Accessing the Docker Host from a Container

The special hostname `host.docker.internal` resolves to the Docker host's IP, which is useful when your container needs to call a service running directly on the host:

```yaml
services:
  api:
    extra_hosts:
      - "host.docker.internal:host-gateway"   # Works on Linux with Docker 20.10+
```

On older Linux Docker versions, find the gateway manually:

```bash
# Get the host IP from inside any container
docker exec -it my-container ip route | grep default | awk '{print $3}'
# Then use that IP in extra_hosts
```

## Setting Extra Hosts via Portainer UI

For containers (not stacks), configure extra hosts in Portainer:

1. Go to **Containers > Add container**.
2. Expand the **Network** tab.
3. Find **Hosts File Entries**.
4. Add hostname and IP pairs.

For existing containers, you must stop and recreate them with the new hosts entries (hosts can't be changed on running containers).

## Dynamic Hosts File Override with a Bind Mount

For more flexibility, mount a custom `hosts` file into the container:

```yaml
services:
  api:
    volumes:
      - ./custom_hosts:/etc/hosts:ro  # Mount a custom hosts file read-only
```

Create `custom_hosts` on the host with your custom entries:

```
127.0.0.1   localhost
::1         localhost ip6-localhost
192.168.1.50   legacy-db
10.0.0.200     payment-gateway
```

This approach lets you update hosts without restarting containers — however, the container's hostname and its own entry in the default hosts file are lost unless you include them manually.

## Common Use Cases

| Use Case | hosts Entry |
|----------|-------------|
| Local SSL testing | `127.0.0.1 myapp.local` |
| Pointing to an older API version | `192.168.1.50 api.internal` |
| Accessing Docker host services | `host.docker.internal host-gateway` |
| Bypassing external DNS for a service | `10.0.0.10 thirdparty-api.com` |
| Testing service migration | `10.0.0.20 database.service` |

## Verifying Hostname Resolution

Check that your custom entries resolve correctly:

```bash
# Test the custom hostname resolves to the right IP
docker exec -it $(docker ps -qf name=api) ping -c 1 legacy-db

# Or use nslookup (will use /etc/hosts before DNS)
docker exec -it $(docker ps -qf name=api) nslookup legacy-db
# Note: nslookup doesn't use /etc/hosts by default; use getent instead
docker exec -it $(docker ps -qf name=api) getent hosts legacy-db
```
