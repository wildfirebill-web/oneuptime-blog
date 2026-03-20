# How to Create a Network Namespace on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Namespaces, Networking, iproute2, Containers, Isolation

Description: Create isolated network namespaces on Linux using the ip netns command to provide separate network stacks for processes and containers.

## Introduction

A network namespace is a logical copy of the Linux network stack, complete with its own interfaces, routing tables, firewall rules, and sockets. Network namespaces are the foundation of container networking (used by Docker, Kubernetes, and LXC). This guide shows how to create and initialize a network namespace manually.

## Prerequisites

- Linux kernel 2.6.24+
- `iproute2` package installed (`ip` command)
- Root or sudo access

## Create a Network Namespace

```bash
# Create a new network namespace named "ns1"

ip netns add ns1

# Verify it was created
ip netns list
```

## Verify the Namespace Was Created

After creation, the namespace appears as a file in `/var/run/netns/`:

```bash
# Check the namespace file
ls -la /var/run/netns/

# The namespace contains only the loopback interface (down by default)
ip netns exec ns1 ip link list
```

## Multiple Namespaces

You can create as many namespaces as needed. Each is completely isolated from the others and from the host:

```bash
# Create multiple namespaces
ip netns add ns1
ip netns add ns2
ip netns add dmz

# List all namespaces
ip netns list
```

## What Is Inside a New Namespace

A freshly created namespace has:
- Only a loopback interface (`lo`) which is DOWN
- An empty routing table
- No connectivity to the outside world

```bash
# Check interfaces in the new namespace
ip netns exec ns1 ip link show

# Check routing table (empty)
ip netns exec ns1 ip route show

# Check if loopback is down
ip netns exec ns1 ip addr show lo
```

## Bring Up the Loopback Interface

```bash
# Enable loopback inside the namespace
ip netns exec ns1 ip link set lo up

# Verify loopback is up
ip netns exec ns1 ip addr show lo
```

## Namespace Identity

Each namespace has its own independent network configuration. Changes in one namespace do not affect others or the host:

```bash
# This only affects ns1 - does not change host routing
ip netns exec ns1 ip route add default via 10.0.0.1

# Host routing is unchanged
ip route show
```

## Namespace with a Custom Name

The namespace name is just a label for the file in `/var/run/netns/`. You can use descriptive names:

```bash
ip netns add web-frontend
ip netns add db-backend
ip netns add monitoring

ip netns list
```

## Conclusion

Creating a network namespace with `ip netns add` provides a completely isolated network environment. New namespaces start with only a down loopback interface and no connectivity - you must explicitly add interfaces and routes. Network namespaces are the building block for container networking, virtual network labs, and network function testing.
