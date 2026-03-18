# How to Fix "network not found" Errors in Podman

Author: [nawazdhandala](https://github.com/nawazdhandala)

Tags: Podman, Containers, Networking, CNI, Troubleshooting

Description: A complete guide to fixing "network not found" errors in Podman, covering network backends, CNI vs Netavark migration, rootless networking, and network configuration files.

---

> Podman's "network not found" error occurs when a container references a network that does not exist, when network configuration files are missing or corrupted, or when switching between CNI and Netavark backends. This guide covers all the common causes and fixes.

Networking in Podman has evolved significantly over the past few years. The transition from CNI (Container Network Interface) plugins to Netavark as the default networking backend introduced new configuration paths and potential compatibility issues. When you see "network not found," the problem could be as simple as a typo in a network name or as complex as a backend migration that left configuration files behind.

This guide walks through each scenario and provides clear solutions.

---

## Understanding Podman Networking Backends

Podman supports two networking backends:

- **CNI** (Container Network Interface): The original backend, using plugins in `/usr/libexec/cni/` or `/opt/cni/bin/` with configuration files in `/etc/cni/net.d/`.
- **Netavark**: The newer default backend (since Podman 4.0+), with its own configuration format stored in Podman's network directory.

Check which backend you are using:

```bash
podman info --format '{{.Host.NetworkBackend}}'
```

The backend determines where network configurations are stored and how they are formatted. Using the wrong paths or formats for your backend is a common source of "network not found" errors.

## Listing Available Networks

Before troubleshooting, check what networks actually exist:

```bash
podman network ls
```

This shows all networks available to your current user. If you are running rootless Podman, you will only see networks created by your user. Rootful networks are stored separately.

For more detail:

```bash
podman network inspect podman
```

The default network is named `podman`. If even this network is missing, your Podman installation has a configuration problem.

## Creating a Missing Network

If the network you need does not exist, create it:

```bash
podman network create mynetwork
```

Create a network with specific options:

```bash
podman network create --subnet 10.89.0.0/24 --gateway 10.89.0.1 mynetwork
```

DNS resolution is enabled by default for custom networks created with Netavark. If you need to explicitly disable it, use the `--disable-dns` flag:

```bash
# DNS is enabled by default, no flag needed
podman network create mynetwork

# To disable DNS resolution
podman network create --disable-dns mynetwork
```

## Rootful vs Rootless Network Stores

Like images, networks are stored separately for rootful and rootless Podman. A network created with `sudo podman network create mynet` is not visible to rootless `podman`.

Rootful network configs (Netavark):
```
/etc/containers/networks/
```

Rootless network configs (Netavark):
```
~/.config/containers/networks/
```

For CNI backend, the paths are:

Rootful:
```
/etc/cni/net.d/
```

Rootless:
```
~/.config/cni/net.d/
```

Verify where your networks are stored:

```bash
# Rootless
ls ~/.config/containers/networks/

# Rootful
sudo ls /etc/containers/networks/
```

## Fixing CNI to Netavark Migration Issues

When upgrading Podman from version 3.x to 4.x or later, the default backend changes from CNI to Netavark. Old CNI network configurations are not automatically migrated. This is one of the most common causes of "network not found" after an upgrade.

Check if you have leftover CNI configs:

```bash
ls /etc/cni/net.d/
ls ~/.config/cni/net.d/
```

If you see network files here but Podman cannot find them, your Podman version is using Netavark instead of CNI.

### Option 1: Switch Back to CNI

Edit the containers configuration to use CNI. For rootless, edit `~/.config/containers/containers.conf`:

```ini
[network]
network_backend = "cni"
```

For rootful, edit `/etc/containers/containers.conf`.

### Option 2: Recreate Networks Under Netavark

Reset and recreate your networks under the new backend:

```bash
# List existing containers and note their network assignments
podman ps -a --format "{{.Names}} {{.Networks}}"

# Remove containers (or stop them first)
podman rm -a -f

# Reset the network configuration
podman system reset

# Recreate your custom networks
podman network create mynetwork --subnet 10.89.0.0/24
```

## Corrupt Network Configuration Files

Network configuration files can become corrupted, especially after system crashes or filesystem issues. If `podman network ls` shows a network but `podman network inspect` fails, the configuration file may be damaged.

For Netavark, inspect the JSON configuration:

```bash
cat ~/.config/containers/networks/mynetwork.json
```

A valid network config looks like:

```json
{
  "name": "mynetwork",
  "id": "abc123...",
  "driver": "bridge",
  "network_interface": "podman1",
  "created": "2026-01-15T10:30:00Z",
  "subnets": [
    {
      "subnet": "10.89.0.0/24",
      "gateway": "10.89.0.1"
    }
  ],
  "ipv6_enabled": false,
  "internal": false,
  "dns_enabled": true
}
```

If the file is corrupted, delete it and recreate the network:

```bash
rm ~/.config/containers/networks/mynetwork.json
podman network create mynetwork
```

## Network Not Found in Compose Files

When using `podman-compose` or `docker-compose` with Podman, network names in the compose file must match existing Podman networks or be defined in the compose file itself.

A common mistake is referencing an external network that does not exist:

```yaml
services:
  web:
    image: nginx
    networks:
      - mynet

networks:
  mynet:
    external: true
```

If `mynet` was not previously created with `podman network create mynet`, this fails. Either create the network first or define it as a non-external network:

```yaml
networks:
  mynet:
    driver: bridge
    ipam:
      config:
        - subnet: 10.89.1.0/24
```

## Network Not Found After System Reboot

Some rootless network configurations may not persist across reboots if they rely on runtime directories. Verify that your network files are in the correct persistent location:

```bash
# These should persist
ls ~/.config/containers/networks/

# This is runtime-only and may be cleared
ls /run/user/$(id -u)/containers/networks/ 2>/dev/null
```

If your networks are in the runtime directory, they will be lost on reboot. Recreate them or ensure they are stored in the config directory.

## Debugging Network Issues

Use verbose logging to see exactly what Podman is searching for:

```bash
podman --log-level=debug network ls 2>&1
```

Check the network backend and configuration paths:

```bash
podman info --format '{{.Host.NetworkBackend}}'
podman info --format '{{.Store.GraphRoot}}'
```

Verify that the required networking tools are installed:

```bash
# For Netavark
which netavark
which aardvark-dns

# For CNI
ls /usr/libexec/cni/ 2>/dev/null || ls /opt/cni/bin/ 2>/dev/null
```

## Conclusion

The "network not found" error in Podman usually stems from one of four issues: the network was never created, it was created in a different context (rootful vs rootless), the networking backend changed during an upgrade (CNI to Netavark), or configuration files are corrupted. Always verify which backend you are running with `podman info`, check that networks exist with `podman network ls`, and use fully qualified network names in compose files. When migrating between Podman versions, plan to recreate custom networks under the new backend.
