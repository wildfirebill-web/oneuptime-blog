# How to Configure DHCP with Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Netplan, Ubuntu, DHCP, IPv4, Networking

Description: Configure DHCP-based IPv4 addressing using Netplan YAML on Ubuntu and Debian systems, with options for DHCP overrides and mixed static/DHCP configurations.

## Introduction

Netplan enables DHCP with `dhcp4: true`. This is the simplest network configuration — the DHCP server assigns the IP address, gateway, and DNS. Additional DHCP options can be controlled via the `dhcp4-overrides` section.

## Basic DHCP Configuration

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
```

```bash
# Apply
netplan apply

# Verify
ip addr show eth0
ip route show
```

## DHCP with Custom Overrides

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        # Send a custom hostname to the DHCP server
        hostname: myserver
        # Use MAC as client identifier
        use-mac: true
        # Do not use DNS from DHCP
        use-dns: false
        # Do not install routes from DHCP
        use-routes: false
      nameservers:
        addresses:
          - 8.8.8.8
```

## DHCP on Multiple Interfaces

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      dhcp4: true
```

## Mixed DHCP and Static on Same Interface

```yaml
network:
  version: 2
  ethernets:
    eth0:
      # DHCP for primary IP
      dhcp4: true
      # Additional static secondary IPs
      addresses:
        - 192.168.1.200/24
```

## DHCP with Route Metric

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        route-metric: 100
```

## Apply and Troubleshoot

```bash
# Apply configuration
netplan apply

# Generate without applying (check for errors)
netplan generate

# Show DHCP lease details
networkctl status eth0
# Or
cat /run/systemd/netif/leases/*
```

## DHCP4-Overrides Reference

| Key | Default | Description |
|---|---|---|
| `use-dns` | true | Accept DNS from DHCP |
| `use-routes` | true | Install routes from DHCP |
| `use-ntp` | true | Accept NTP servers |
| `hostname` | system hostname | Hostname sent to server |
| `send-hostname` | true | Whether to send hostname |
| `use-mac` | false | Use MAC as client identifier |
| `route-metric` | 100 | Metric for DHCP-assigned routes |

## Conclusion

DHCP in Netplan is as simple as `dhcp4: true`. Use `dhcp4-overrides` to control what DHCP options are accepted. Apply with `netplan apply`, and verify with `ip addr show`. Use `netplan try` to test changes safely before committing.
