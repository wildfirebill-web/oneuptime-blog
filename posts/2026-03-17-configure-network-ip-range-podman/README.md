# How to Configure Network IP Range in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Networking, IP Range, Subnets

Description: Learn how to configure custom IP address ranges and subnets when creating Podman networks.

---

> Defining a custom IP range for your Podman network prevents address collisions and gives you predictable container addressing.

By default, Podman assigns subnets from an internal pool. This can conflict with your host network, VPN ranges, or other container networks. Setting an explicit subnet and gateway when creating a network ensures containers receive addresses in a range you control.

---

## Creating a Network with a Custom Subnet

```bash
# Create a network with a specific subnet and gateway
podman network create \
  --subnet 10.50.0.0/24 \
  --gateway 10.50.0.1 \
  custom-range-net

# Inspect the network to verify the configuration
podman network inspect custom-range-net
```

## Assigning a Static IP to a Container

Once you have a network with a known subnet, you can assign static IPs to containers.

```bash
# Run a container with a static IP in the custom range
podman run -d --name webserver \
  --network custom-range-net \
  --ip 10.50.0.10 \
  docker.io/library/nginx:alpine

# Verify the assigned IP
podman inspect webserver --format '{{.NetworkSettings.Networks}}'
```

## Using a Larger Subnet

```bash
# Create a network with a /16 subnet for large deployments
podman network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  large-net

# Containers will receive IPs from 172.20.0.2 to 172.20.255.254
podman run -d --name app1 --network large-net docker.io/library/alpine sleep 3600
podman run -d --name app2 --network large-net docker.io/library/alpine sleep 3600

# Check their assigned addresses
podman exec app1 ip addr show eth0
podman exec app2 ip addr show eth0
```

## Creating an IPv6 Network

```bash
# Create a dual-stack network with both IPv4 and IPv6
podman network create \
  --subnet 10.60.0.0/24 \
  --gateway 10.60.0.1 \
  --ipv6 \
  --subnet fd00:10:60::/64 \
  --gateway fd00:10:60::1 \
  dualstack-net

# Run a container on the dual-stack network
podman run --rm --network dualstack-net docker.io/library/alpine ip addr show eth0
```

## Avoiding Subnet Conflicts

```bash
# List all existing Podman networks and their subnets
podman network ls
podman network inspect --format '{{.Subnets}}' $(podman network ls -q)

# Check host routing table to find ranges already in use
ip route show

# Choose a subnet that does not overlap with any existing range
podman network create \
  --subnet 10.99.0.0/24 \
  --gateway 10.99.0.1 \
  isolated-net
```

## Summary

Configuring custom IP ranges in Podman networks gives you control over address assignment, prevents conflicts with host or VPN subnets, and enables static IP allocation for containers that need predictable addresses.
