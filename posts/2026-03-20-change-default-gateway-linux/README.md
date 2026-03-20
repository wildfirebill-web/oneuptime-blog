# How to Change the Default Gateway on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Default Gateway, Routing, iproute2, Networking, Network Configuration

Description: Change the existing default gateway on Linux using ip route commands to redirect all outbound traffic through a new gateway without disrupting active connections.

## Introduction

Changing the default gateway requires removing the old route and adding a new one. On production systems, this should be done carefully to minimize disruption. The `ip route replace` command provides an atomic operation that changes the gateway in one step.

## Method 1: Replace (Atomic, Preferred)

```bash
# Replace replaces an existing route atomically
# This is the safest method — no gap in connectivity
ip route replace default via 192.168.1.254
```

## Method 2: Delete and Add

```bash
# First check the current default gateway
ip route show default
# default via 192.168.1.1 dev eth0

# Remove the old default gateway
ip route del default via 192.168.1.1

# Add the new default gateway
ip route add default via 192.168.1.254

# Verify
ip route show default
```

## Method 3: Add with Lower Metric (Non-Disruptive Testing)

```bash
# Add new gateway with a lower metric (preferred over existing)
ip route add default via 192.168.1.254 metric 50

# Existing gateway stays but new one has priority (metric 100 default)
ip route show default

# If new gateway works, remove the old one
ip route del default via 192.168.1.1 metric 100
```

## Change Gateway via nmcli (RHEL, Persistent)

```bash
# Modify the gateway in the connection profile
nmcli connection modify eth0 ipv4.gateway "192.168.1.254"

# Apply the change
nmcli connection up eth0

# Verify
ip route show default
```

## Change Gateway via Netplan (Ubuntu, Persistent)

```yaml
# /etc/netplan/01-network.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.100/24]
      routes:
        - to: default
          via: 192.168.1.254   # Changed from .1 to .254
```

```bash
netplan apply
```

## Verify the New Gateway

```bash
# Check the routing table
ip route show default

# Test connectivity through the new gateway
ping -c 3 192.168.1.254

# Test internet access
ping -c 3 8.8.8.8

# Trace route to confirm traffic goes through new gateway
traceroute 8.8.8.8
# First hop should now be 192.168.1.254
```

## Handle Multiple Default Routes

```bash
# List all default routes
ip route show 0.0.0.0/0

# If multiple exist, they compete based on metric
# Lower metric wins
```

## Conclusion

Use `ip route replace default via <new-gateway>` for the safest and most atomic gateway change. This command replaces the existing route without a gap in connectivity. For persistent changes, update the gateway in your distribution's network configuration tool (nmcli, Netplan, systemd-networkd). Always verify with `ping` through the new gateway before assuming the change is successful.
