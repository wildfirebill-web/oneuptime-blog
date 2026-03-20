# How to Set Up DNS Resolution Inside a Network Namespace

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Namespaces, DNS, resolv.conf, Networking, Containers

Description: Configure DNS resolution inside a Linux network namespace using /etc/netns/ to provide namespace-specific resolv.conf files for name resolution.

## Introduction

Network namespaces have isolated network stacks, but by default they share the host's `/etc/resolv.conf`. Linux provides a mechanism to give each namespace its own DNS configuration using the `/etc/netns/<namespace-name>/resolv.conf` file, which overrides the global file for processes in that namespace.

## Prerequisites

- A configured network namespace with internet access
- Root access

## The Default Behavior

Without namespace-specific DNS configuration, the namespace uses the host's `/etc/resolv.conf`:

```bash
# Check DNS resolution in the namespace (uses host resolv.conf by default)
ip netns exec ns1 cat /etc/resolv.conf
# Shows the host's resolv.conf
```

## Create a Namespace-Specific resolv.conf

Linux reads `/etc/netns/<namespace>/resolv.conf` when executing commands inside named namespaces:

```bash
# Create the namespace directory
mkdir -p /etc/netns/ns1

# Create the DNS configuration for ns1
cat > /etc/netns/ns1/resolv.conf << 'EOF'
# DNS servers for namespace ns1
nameserver 8.8.8.8
nameserver 8.8.4.4
search example.internal
EOF

# Verify the namespace now uses its own DNS config
ip netns exec ns1 cat /etc/resolv.conf
```

## Test DNS Resolution

```bash
# Test name resolution inside the namespace
ip netns exec ns1 nslookup google.com

# Test with dig (if available)
ip netns exec ns1 dig +short google.com

# Test with ping (uses DNS)
ip netns exec ns1 ping -c 2 google.com
```

## Use a Local DNS Resolver

For advanced setups, run a DNS resolver (like dnsmasq) inside the namespace and point `resolv.conf` to it:

```bash
# Start dnsmasq inside the namespace listening on 127.0.0.1
ip netns exec ns1 dnsmasq --no-daemon --interface=lo \
    --bind-interfaces --listen-address=127.0.0.1 \
    --server=8.8.8.8 &

# Configure resolv.conf to use the local resolver
cat > /etc/netns/ns1/resolv.conf << 'EOF'
nameserver 127.0.0.1
EOF
```

## DNS with systemd-resolved

If the host uses systemd-resolved, configure per-namespace DNS differently:

```bash
# Use a direct DNS server (bypass systemd-resolved stub)
cat > /etc/netns/ns1/resolv.conf << 'EOF'
nameserver 1.1.1.1
nameserver 9.9.9.9
EOF
```

## Custom Search Domains

```bash
# Namespace with custom search domain (for internal services)
cat > /etc/netns/prod/resolv.conf << 'EOF'
nameserver 10.0.0.53
search prod.internal corp.internal
options ndots:3
EOF
```

## Full Setup Script Including DNS

```bash
#!/bin/bash
# ns-with-dns.sh: Create a namespace with internet access and DNS

NS="ns1"
NS_DNS_DIR="/etc/netns/$NS"

# Create namespace and configure networking
ip netns add $NS
ip link add veth-host type veth peer name veth-ns
ip link set veth-ns netns $NS
ip addr add 10.0.0.1/24 dev veth-host && ip link set veth-host up
ip netns exec $NS ip link set lo up
ip netns exec $NS ip addr add 10.0.0.2/24 dev veth-ns
ip netns exec $NS ip link set veth-ns up
ip netns exec $NS ip route add default via 10.0.0.1

# Enable NAT
sysctl -w net.ipv4.ip_forward=1 > /dev/null
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE 2>/dev/null

# Configure DNS
mkdir -p $NS_DNS_DIR
cat > $NS_DNS_DIR/resolv.conf << 'EOF'
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

echo "Testing DNS resolution in $NS..."
ip netns exec $NS ping -c 2 google.com && echo "DNS working!"
```

## Conclusion

Per-namespace DNS configuration is achieved through `/etc/netns/<name>/resolv.conf`. Linux automatically uses this file instead of `/etc/resolv.conf` when executing commands inside named namespaces with `ip netns exec`. This enables each namespace to have independent DNS servers and search domains — matching how container orchestrators configure per-container DNS.
