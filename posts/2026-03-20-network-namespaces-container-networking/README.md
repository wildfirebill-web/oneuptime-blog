# How to Use Network Namespaces for Container Networking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Network Namespaces, Container Networking, Docker, Linux, veth, Bridge, IPv4

Description: Learn how Linux network namespaces underpin container networking, and how to manually replicate what Docker does to connect containers to a bridge network with IPv4 connectivity.

---

Docker and Kubernetes use Linux network namespaces to give each container its own network stack. Understanding the underlying mechanics helps with debugging and building custom network solutions.

## How Docker Uses Namespaces

```text
Host                         Container (namespace: ns-c1)
  docker0 (172.17.0.1)          eth0 (172.17.0.2)
      |                              |
  veth123abc (host side)      veth456def (container side, moved into ns)
      └──────────────────────────────┘
                  veth pair
```

## Replicating Docker Bridge Networking Manually

### Step 1: Create a Bridge

```bash
ip link add docker0 type bridge
ip addr add 172.17.0.1/16 dev docker0
ip link set docker0 up
```

### Step 2: Create a Network Namespace (Container)

```bash
ip netns add container1
```

### Step 3: Create veth Pair and Connect to Bridge

```bash
# Create veth pair

ip link add veth-host type veth peer name veth-cont

# Move container-side into namespace
ip link set veth-cont netns container1

# Connect host side to bridge
ip link set veth-host master docker0
ip link set veth-host up
```

### Step 4: Configure the Container Namespace

```bash
# Assign IP inside namespace
ip netns exec container1 ip addr add 172.17.0.2/16 dev veth-cont
ip netns exec container1 ip link set veth-cont up
ip netns exec container1 ip link set lo up

# Add default route
ip netns exec container1 ip route add default via 172.17.0.1
```

### Step 5: Enable NAT for Internet Access

```bash
# Enable forwarding
sysctl -w net.ipv4.ip_forward=1

# Masquerade outbound traffic from containers
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

# Allow forwarding between bridge and external interface
iptables -A FORWARD -i docker0 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o docker0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### Step 6: Test Connectivity

```bash
# From container to host
ip netns exec container1 ping 172.17.0.1

# From container to internet
ip netns exec container1 ping 8.8.8.8
```

## Inspecting Docker Namespaces

```bash
# Find the network namespace of a running container
docker inspect container_id | jq '.[].NetworkSettings.SandboxKey'
# Returns: /var/run/docker/netns/abc123

# Execute in that namespace
nsenter --net=/var/run/docker/netns/abc123 ip addr show
```

## Key Takeaways

- Docker creates a network namespace per container, moves a veth pair endpoint into it, and connects the host side to the docker0 bridge.
- IP forwarding + iptables MASQUERADE is what enables containers to reach the internet.
- Understanding this mechanism helps debug container networking issues and build custom CNI plugins.
- Use `nsenter --net=<sandbox-key>` to inspect a running container's network namespace without exec'ing into the container.
