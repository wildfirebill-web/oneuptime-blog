# How to Connect Network Namespaces Using veth Pairs on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, Network Namespaces, veth, IPv4, Container

Description: Connect two Linux network namespaces using a virtual Ethernet (veth) pair, assign IPv4 addresses on each end, and verify bidirectional connectivity between isolated network stacks.

## Introduction

A veth (virtual Ethernet) pair is a linked pair of virtual interfaces - packets entering one end immediately emerge from the other. By placing each end in a different network namespace, you create a direct Layer 2 connection between the two isolated stacks.

## Create Two Namespaces

```bash
# Create two isolated namespaces

sudo ip netns add ns1
sudo ip netns add ns2

# Verify
ip netns list
```

## Create a veth Pair

```bash
# Create a veth pair: veth-ns1 and veth-ns2
sudo ip link add veth-ns1 type veth peer name veth-ns2
```

At this point both interfaces are in the host namespace. Verify:

```bash
ip link show type veth
# Shows both veth-ns1 and veth-ns2
```

## Move Each End into Its Namespace

```bash
# Move veth-ns1 into ns1
sudo ip link set veth-ns1 netns ns1

# Move veth-ns2 into ns2
sudo ip link set veth-ns2 netns ns2
```

Now neither interface is visible in the host namespace - they live inside their respective namespaces.

## Assign IPv4 Addresses

```bash
# Assign IP and bring up veth inside ns1
sudo ip netns exec ns1 ip addr add 10.1.1.1/24 dev veth-ns1
sudo ip netns exec ns1 ip link set veth-ns1 up
sudo ip netns exec ns1 ip link set lo up

# Assign IP and bring up veth inside ns2
sudo ip netns exec ns2 ip addr add 10.1.1.2/24 dev veth-ns2
sudo ip netns exec ns2 ip link set veth-ns2 up
sudo ip netns exec ns2 ip link set lo up
```

## Verify Connectivity

```bash
# Ping ns2 from ns1
sudo ip netns exec ns1 ping -c 3 10.1.1.2

# Ping ns1 from ns2
sudo ip netns exec ns2 ping -c 3 10.1.1.1
```

Both should show 0% packet loss.

## Opening a Shell in Each Namespace

```bash
# Open shells in each namespace in separate terminals
sudo ip netns exec ns1 bash
sudo ip netns exec ns2 bash
```

Each shell sees only the interfaces and routes in its namespace.

## Practical Use: Testing Network Applications

```bash
# Start a server in ns2
sudo ip netns exec ns2 python3 -m http.server 8080 &

# Connect from ns1
sudo ip netns exec ns1 curl http://10.1.1.2:8080
```

## Automating with a Script

```bash
#!/bin/bash
# Set up two connected namespaces with IPv4

NS1=ns1; NS2=ns2; V1=veth-ns1; V2=veth-ns2
IP1=10.1.1.1/24; IP2=10.1.1.2/24

sudo ip netns add $NS1
sudo ip netns add $NS2
sudo ip link add $V1 type veth peer name $V2
sudo ip link set $V1 netns $NS1
sudo ip link set $V2 netns $NS2
sudo ip netns exec $NS1 bash -c "ip link set lo up; ip addr add $IP1 dev $V1; ip link set $V1 up"
sudo ip netns exec $NS2 bash -c "ip link set lo up; ip addr add $IP2 dev $V2; ip link set $V2 up"
echo "Namespaces connected: $NS1($IP1) ↔ $NS2($IP2)"
```

## Cleanup

```bash
# Delete namespaces (veth interfaces inside are automatically removed)
sudo ip netns del ns1
sudo ip netns del ns2
```

## Conclusion

veth pairs are the simplest way to connect two network namespaces. Each namespace sees only its own end of the pair. This pattern is used by Docker, Kubernetes, and systemd-nspawn to connect container namespaces to bridge interfaces on the host.
