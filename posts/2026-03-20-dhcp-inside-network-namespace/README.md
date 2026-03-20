# How to Set Up DHCP Inside a Network Namespace

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Network Namespace, Linux, netns, dnsmasq, IPv4, Testing

Description: Learn how to run a DHCP server inside a Linux network namespace to provide IP addresses to interfaces within the namespace for isolated network testing and container networking.

---

Running DHCP inside a network namespace enables isolated IP management for test environments, container networking simulation, and multi-tenant setups where each namespace needs its own DHCP service.

## Architecture

```
Host
  └── veth-host (10.0.100.1/24)
        |
      Bridge / veth pair
        |
  Namespace: dhcp-ns
  └── veth-ns (DHCP server: 10.0.100.1)
       └── dnsmasq serves 10.0.100.50-200

  Namespace: client-ns
  └── veth-client (DHCP client: gets 10.0.100.50)
```

## Step 1: Create Namespaces and veth Pairs

```bash
# Create namespaces
ip netns add dhcp-ns
ip netns add client-ns

# veth pair: host to dhcp-ns
ip link add veth-dhcp type veth peer name veth-dhcp-ns
ip link set veth-dhcp-ns netns dhcp-ns

# veth pair: dhcp-ns to client-ns (simulating a shared segment)
ip link add veth-server type veth peer name veth-client
ip link set veth-server netns dhcp-ns
ip link set veth-client netns client-ns
```

## Step 2: Configure the DHCP Server Namespace

```bash
# Assign IP to DHCP server interface
ip netns exec dhcp-ns ip addr add 10.0.100.1/24 dev veth-server
ip netns exec dhcp-ns ip link set veth-server up
ip netns exec dhcp-ns ip link set lo up
```

## Step 3: Run dnsmasq Inside the Namespace

```bash
# Start dnsmasq in dhcp-ns
ip netns exec dhcp-ns dnsmasq \
  --interface=veth-server \
  --bind-interfaces \
  --dhcp-range=10.0.100.50,10.0.100.200,255.255.255.0,1h \
  --dhcp-option=3,10.0.100.1 \
  --dhcp-option=6,8.8.8.8 \
  --no-resolv \
  --pid-file=/run/dnsmasq-dhcp-ns.pid \
  --log-facility=/var/log/dnsmasq-dhcp-ns.log

# Verify it's running
ip netns exec dhcp-ns ss -ulnp | grep dnsmasq
```

## Step 4: Get DHCP Lease in Client Namespace

```bash
# Bring up client interface
ip netns exec client-ns ip link set veth-client up
ip netns exec client-ns ip link set lo up

# Request DHCP lease
ip netns exec client-ns dhclient veth-client

# Verify assigned address
ip netns exec client-ns ip addr show veth-client
ip netns exec client-ns ip route show
```

## Step 5: Verify Connectivity

```bash
# Ping DHCP server from client namespace
ip netns exec client-ns ping 10.0.100.1

# View DHCP leases
ip netns exec dhcp-ns cat /var/lib/misc/dnsmasq.leases
```

## Cleanup

```bash
# Release DHCP lease
ip netns exec client-ns dhclient -r veth-client

# Stop dnsmasq
kill $(cat /run/dnsmasq-dhcp-ns.pid)

# Delete namespaces (removes all interfaces)
ip netns del dhcp-ns
ip netns del client-ns
```

## Key Takeaways

- Run dnsmasq with `--interface` and `--bind-interfaces` to serve DHCP only on the namespace's interface.
- Each network namespace is fully isolated: DHCP in one namespace doesn't affect others.
- Use `ip netns exec <ns> dhclient <iface>` to request a lease inside a namespace.
- Namespace-based DHCP testing is useful for validating dnsmasq configuration before deploying to production.
