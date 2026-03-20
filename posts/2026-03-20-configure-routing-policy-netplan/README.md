# How to Configure Routing Policy with Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Netplan, Ubuntu, Policy Routing, Routing, Networking

Description: Configure policy-based routing rules with Netplan YAML using routing-policy entries, enabling different traffic flows based on source address, mark, or table.

## Introduction

Policy-based routing in Netplan uses the `routing-policy` key under each interface. Rules define conditions (from, mark, type-of-service) and which routing table to use. This enables multi-homing, VPN split tunneling, and traffic engineering.

## Basic Policy Routing Rule

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.0.0.10/24
      routes:
        - to: default
          via: 10.0.0.1
          table: 100
        - to: 10.0.0.0/24
          via: 10.0.0.1
          table: 100
      routing-policy:
        - from: 10.0.0.10/32
          table: 100
          priority: 100
```

```bash
netplan apply
ip rule list
ip route show table 100
```

## Multi-Homing with Two ISPs

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.0.0.10/24
      routes:
        - to: default
          via: 10.0.0.1
          table: 100
        - to: 10.0.0.0/24
          via: 10.0.0.1
          table: 100
      routing-policy:
        - from: 10.0.0.10/32
          table: 100

    eth1:
      dhcp4: false
      addresses:
        - 10.0.1.10/24
      routes:
        - to: default
          via: 10.0.1.1
          table: 200
        - to: 10.0.1.0/24
          via: 10.0.1.1
          table: 200
      routing-policy:
        - from: 10.0.1.10/32
          table: 200
```

## routing-policy Fields

```yaml
routing-policy:
  - from: 192.168.1.0/24   # Match source IP/subnet
    to: 0.0.0.0/0           # Match destination (optional)
    table: 100               # Use this routing table
    priority: 100            # Rule priority (lower = checked first)
    mark: 1                  # Match by fwmark (optional)
    type-of-service: 16      # Match by ToS (optional)
```

## Route to Specific Table

```yaml
routes:
  - to: default
    via: 10.0.0.1
    # This route goes into table 100 instead of main
    table: 100
    metric: 100
```

## Verify Policy Routing

```bash
# Apply
netplan apply

# Show all routing rules
ip rule list

# Show routes in custom table
ip route show table 100

# Test which rule applies
ip route get 8.8.8.8 from 10.0.0.10
```

## Conclusion

Netplan routing policy uses `routing-policy` (for ip rules) and `routes` with `table` (for routes in custom tables). Each rule specifies a match condition (`from`, `mark`) and a target table. Apply with `netplan apply` and verify with `ip rule list` and `ip route show table <number>`.
