# How to Create a veth Pair with ip link add type veth

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, ip command, iproute2, veth, Network Namespaces, Containers, Networking

Description: Create virtual Ethernet (veth) pairs using ip link add type veth to connect network namespaces, containers, bridges, or virtual machines.

## Introduction

A veth (virtual Ethernet) pair is two interconnected virtual network interfaces. Packets sent into one end come out the other end — they act as a pipe. veth pairs are the building block for container networking, network namespace connectivity, and virtual machine network bridging.

## Create a veth Pair

```bash
# Create a veth pair: veth0 and veth1 are connected
ip link add veth0 type veth peer name veth1

# Both interfaces exist in the same namespace initially
ip link show veth0
ip link show veth1
```

## Assign IPs and Bring Up

```bash
# Assign IPs to each end
ip addr add 10.0.0.1/24 dev veth0
ip addr add 10.0.0.2/24 dev veth1

# Bring both up
ip link set veth0 up
ip link set veth1 up

# Test connectivity
ping -c 3 10.0.0.2
```

## Connect Two Network Namespaces with veth

```bash
# Create two namespaces
ip netns add ns1
ip netns add ns2

# Create veth pair
ip link add veth-ns1 type veth peer name veth-ns2

# Move each end to its namespace
ip link set veth-ns1 netns ns1
ip link set veth-ns2 netns ns2

# Configure each end
ip netns exec ns1 ip addr add 10.10.0.1/24 dev veth-ns1
ip netns exec ns1 ip link set veth-ns1 up
ip netns exec ns1 ip link set lo up

ip netns exec ns2 ip addr add 10.10.0.2/24 dev veth-ns2
ip netns exec ns2 ip link set veth-ns2 up
ip netns exec ns2 ip link set lo up

# Test connectivity
ip netns exec ns1 ping -c 3 10.10.0.2
```

## Connect Namespace to a Bridge

```bash
# Create veth pair
ip link add veth0 type veth peer name veth0-br

# Attach one end to bridge
ip link set veth0-br master br0
ip link set veth0-br up

# Move other end to namespace
ip link set veth0 netns myns
ip netns exec myns ip addr add 192.168.1.10/24 dev veth0
ip netns exec myns ip link set veth0 up
```

## Show veth Peer Information

```bash
# Show which interface is paired with which
ip link show veth0

# Output includes: link-netnsid (if peer is in another ns)
# Or see the peer index number
```

## Delete a veth Pair

```bash
# Deleting one end deletes both
ip link delete veth0

# Verify both are gone
ip link show | grep veth
```

## Conclusion

`ip link add <name> type veth peer name <peer-name>` creates a connected pair of virtual interfaces. Use them to link network namespaces, attach containers to bridges, or build virtual network topologies. Deleting one end of the pair automatically removes both interfaces.
