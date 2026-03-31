# How to Create a veth Pair Between Two Network Namespaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Namespaces, veth, iproute2, Networking, Container, Virtual Networking

Description: Create virtual ethernet (veth) pairs and move each end into separate network namespaces to establish direct connectivity between isolated network environments.

## Introduction

A veth (virtual ethernet) pair is a pair of virtual network interfaces that are connected like a pipe - packets sent into one end come out the other. By placing each end of a veth pair in a different network namespace, you create a point-to-point link between two otherwise isolated network environments. This is how container runtimes connect containers to a host bridge.

## Prerequisites

- Two network namespaces created (or use the host as one end)
- Root access
- iproute2 installed

## Step 1: Create Two Network Namespaces

```bash
# Create two namespaces

ip netns add ns1
ip netns add ns2

# Verify they exist
ip netns list
```

## Step 2: Create a veth Pair

Create the veth pair on the host - both interfaces start in the host (default) namespace:

```bash
# Create a veth pair: veth0 and veth1 are linked
ip link add veth0 type veth peer name veth1

# Verify both interfaces are visible on the host
ip link show type veth
```

## Step 3: Move Each End Into a Namespace

```bash
# Move veth0 into ns1
ip link set veth0 netns ns1

# Move veth1 into ns2
ip link set veth1 netns ns2

# Verify: neither veth0 nor veth1 should appear on the host anymore
ip link show type veth
```

## Step 4: Configure IP Addresses Inside Each Namespace

```bash
# Configure ns1 side
ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth0
ip netns exec ns1 ip link set veth0 up
ip netns exec ns1 ip link set lo up

# Configure ns2 side
ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth1
ip netns exec ns2 ip link set veth1 up
ip netns exec ns2 ip link set lo up
```

## Step 5: Test Connectivity

```bash
# Ping ns2 from ns1
ip netns exec ns1 ping -c 3 10.0.0.2

# Ping ns1 from ns2
ip netns exec ns2 ping -c 3 10.0.0.1
```

## Full Setup Script

```bash
#!/bin/bash
# setup-veth-namespaces.sh
# Creates two namespaces connected by a veth pair

set -e

# Create namespaces
ip netns add ns1
ip netns add ns2

# Create veth pair
ip link add veth0 type veth peer name veth1

# Move interfaces into namespaces
ip link set veth0 netns ns1
ip link set veth1 netns ns2

# Configure ns1
ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth0
ip netns exec ns1 ip link set veth0 up
ip netns exec ns1 ip link set lo up

# Configure ns2
ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth1
ip netns exec ns2 ip link set veth1 up
ip netns exec ns2 ip link set lo up

echo "Setup complete. Testing connectivity..."
ip netns exec ns1 ping -c 3 10.0.0.2 && echo "SUCCESS: ns1 can reach ns2"
```

## veth Pair with Host as One End

To connect a namespace to the host network stack:

```bash
# Keep veth0 on the host, move veth1 into ns1
ip link add veth0 type veth peer name veth1
ip link set veth1 netns ns1

# Configure the host side
ip addr add 10.0.1.1/24 dev veth0
ip link set veth0 up

# Configure the namespace side
ip netns exec ns1 ip addr add 10.0.1.2/24 dev veth1
ip netns exec ns1 ip link set veth1 up
```

## Conclusion

veth pairs are the fundamental building block for connecting network namespaces. Create the pair on the host, then move each end into the appropriate namespace. Once IPs are assigned and interfaces are brought up, the two namespaces can communicate directly. This pattern is used by Docker (with a bridge) and by many virtual network lab setups.
