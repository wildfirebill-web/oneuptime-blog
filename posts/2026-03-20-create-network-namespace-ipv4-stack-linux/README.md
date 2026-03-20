# How to Create a Network Namespace with Its Own IPv4 Stack on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, Network Namespaces, IPv4, Containers, iproute2

Description: Create an isolated Linux network namespace with its own IPv4 stack, loopback interface, and routing table, and run processes inside it using ip netns and nsenter.

## Introduction

Linux network namespaces provide complete isolation of the network stack - each namespace has its own interfaces, routing table, iptables rules, and sockets. This is the foundation of container networking in Docker and Kubernetes.

## Creating a Network Namespace

```bash
# Create a named network namespace

sudo ip netns add myns

# List all namespaces
ip netns list
# Output: myns
```

The namespace is empty - it has only a loopback interface and no routing.

## Bringing Up Loopback in the Namespace

```bash
# Execute a command inside the namespace
sudo ip netns exec myns ip link show
# Shows only: 1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN

# Bring up loopback
sudo ip netns exec myns ip link set lo up

# Verify
sudo ip netns exec myns ip link show lo
# Now shows: state UNKNOWN (loopback up)
```

## Creating a Virtual Interface Pair and Moving One End

To give the namespace internet access, create a veth pair:

```bash
# Create veth pair: veth0 stays in host, veth1 goes to namespace
sudo ip link add veth0 type veth peer name veth1

# Move veth1 into the namespace
sudo ip link set veth1 netns myns

# Verify
sudo ip netns exec myns ip link show
# Shows: lo and veth1
```

## Assigning IPv4 Addresses

```bash
# Assign IP to host end (veth0)
sudo ip addr add 10.0.0.1/24 dev veth0
sudo ip link set veth0 up

# Assign IP to namespace end (veth1)
sudo ip netns exec myns ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec myns ip link set veth1 up
```

## Adding a Default Route in the Namespace

```bash
# Route traffic out via the host (10.0.0.1)
sudo ip netns exec myns ip route add default via 10.0.0.1

# Verify routing table inside namespace
sudo ip netns exec myns ip route show
```

## Testing Connectivity

```bash
# Ping from host to namespace
ping -c 3 10.0.0.2

# Ping from namespace to host
sudo ip netns exec myns ping -c 3 10.0.0.1

# Run a shell inside the namespace
sudo ip netns exec myns bash
# Inside: ip addr, ping, etc. - all isolated
```

## Running a Service Inside the Namespace

```bash
# Start a web server bound to the namespace IP
sudo ip netns exec myns python3 -m http.server 8080 &

# Test from the host
curl http://10.0.0.2:8080
```

## Deleting a Namespace

```bash
# Delete the namespace (also removes all interfaces inside it)
sudo ip netns del myns
```

## Conclusion

Network namespaces provide complete IPv4 stack isolation at the process level. Create with `ip netns add`, populate with veth pairs and IP addresses, and execute commands with `ip netns exec`. This is the same mechanism Docker and Kubernetes use under the hood for container networking.
