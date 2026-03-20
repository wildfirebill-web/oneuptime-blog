# How to Persist Network Namespace Configuration Across Reboots

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Namespaces, Systemd, Persistence, Networking, Automation

Description: Make network namespace configurations survive reboots by creating systemd service units that recreate namespaces, interfaces, and routes at boot time.

## Introduction

Network namespaces created with `ip netns add` are not persistent - they disappear on reboot. To retain your namespace topology across reboots, you need to automate the setup using systemd service units or network management tools like systemd-networkd.

## Prerequisites

- systemd-based Linux distribution
- Root access
- A working namespace configuration you want to persist

## Method 1: systemd Service Unit

Create a shell script with all the setup commands and a systemd service to run it at boot.

### Step 1: Create the Setup Script

```bash
cat > /usr/local/bin/setup-namespaces.sh << 'EOF'
#!/bin/bash
# setup-namespaces.sh: Recreate network namespaces at boot

set -e

# Create namespace

ip netns add ns1 2>/dev/null || true

# Create veth pair
ip link add veth-host type veth peer name veth-ns 2>/dev/null || true

# Move namespace side into ns1
ip link set veth-ns netns ns1 2>/dev/null || true

# Configure host side
ip addr add 10.0.0.1/24 dev veth-host 2>/dev/null || true
ip link set veth-host up

# Configure namespace side
ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth-ns 2>/dev/null || true
ip netns exec ns1 ip link set veth-ns up
ip netns exec ns1 ip route add default via 10.0.0.1 2>/dev/null || true

# Enable NAT for internet access
sysctl -w net.ipv4.ip_forward=1 > /dev/null
iptables -t nat -C POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE 2>/dev/null || \
    iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

echo "Namespace ns1 configured"
EOF

chmod +x /usr/local/bin/setup-namespaces.sh
```

### Step 2: Create the Teardown Script

```bash
cat > /usr/local/bin/teardown-namespaces.sh << 'EOF'
#!/bin/bash
# teardown-namespaces.sh: Clean up namespaces on shutdown

ip netns delete ns1 2>/dev/null || true
ip link delete veth-host 2>/dev/null || true
iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE 2>/dev/null || true
EOF

chmod +x /usr/local/bin/teardown-namespaces.sh
```

### Step 3: Create the systemd Service Unit

```bash
cat > /etc/systemd/system/network-namespaces.service << 'EOF'
[Unit]
Description=Custom Network Namespaces Setup
After=network.target
Wants=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/setup-namespaces.sh
ExecStop=/usr/local/bin/teardown-namespaces.sh

[Install]
WantedBy=multi-user.target
EOF
```

### Step 4: Enable and Start the Service

```bash
systemctl daemon-reload
systemctl enable network-namespaces.service
systemctl start network-namespaces.service

# Verify
systemctl status network-namespaces.service
ip netns list
```

## Method 2: Using systemd-networkd .netdev Files

For `systemd-networkd`-managed systems, virtual devices can be defined in `.netdev` files. While full namespace support is limited in networkd, it handles veth pairs and bridges well.

## Persist DNS Configuration

```bash
# Create the DNS config directory and file
mkdir -p /etc/netns/ns1
cat > /etc/netns/ns1/resolv.conf << 'EOF'
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF
# This file persists automatically (it's on the filesystem)
```

## Verify After Reboot

```bash
# After reboot, check the service started correctly
systemctl status network-namespaces.service
journalctl -u network-namespaces.service

# Verify namespace is up
ip netns list
ip netns exec ns1 ip addr show
ip netns exec ns1 ping -c 2 8.8.8.8
```

## Conclusion

Network namespaces do not persist by default. Use a systemd `oneshot` service with `RemainAfterExit=yes` to run your setup script at boot and a teardown script at shutdown. Store the setup script in `/usr/local/bin/`, place the systemd unit in `/etc/systemd/system/`, and enable it with `systemctl enable`. This pattern is reliable and integrates cleanly with systemd's dependency management.
