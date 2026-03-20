# How to Configure Static Routes with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, Static Routes, Routing, Networking, Configuration

Description: Configure persistent static routes using systemd-networkd .network files, including host routes, network routes, default routes, and metrics.

## Introduction

Static routes in systemd-networkd are defined in `[Route]` sections within `.network` files. Routes are automatically applied when the interface comes up and are removed when the interface goes down. This provides clean lifecycle management without startup scripts.

## Basic Static Route

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=10.0.0.10/24
Gateway=10.0.0.1

[Route]
# Route to 192.168.50.0/24 via a specific gateway
Destination=192.168.50.0/24
Gateway=10.0.0.2
```

## Multiple Static Routes

```ini
[Match]
Name=eth0

[Network]
Address=10.0.0.10/24

[Route]
Destination=192.168.50.0/24
Gateway=10.0.0.2

[Route]
Destination=172.16.0.0/16
Gateway=10.0.0.3

[Route]
Destination=10.20.0.0/24
Gateway=10.0.0.4
Metric=200
```

## Default Gateway Route

```ini
[Route]
# Default route (all traffic not matched by other routes)
Destination=0.0.0.0/0
Gateway=10.0.0.1
```

The `Gateway=` key in `[Network]` also sets the default route, but `[Route]` with `Destination=0.0.0.0/0` provides more control.

## Route with Metric

```ini
[Route]
Destination=192.168.50.0/24
Gateway=10.0.0.2
# Lower metric = higher priority
Metric=100
```

## Host Route (Single IP)

```ini
[Route]
Destination=10.5.5.5/32
Gateway=10.0.0.2
```

## Route with Source Address

```ini
[Route]
Destination=192.168.50.0/24
Gateway=10.0.0.2
# Prefer packets sourced from this IP
PreferredSource=10.0.0.10
```

## Blackhole Route

```ini
[Route]
Destination=10.99.0.0/24
Type=blackhole
```

## Apply and Verify

```bash
# Reload configuration
networkctl reload

# Or restart the service
systemctl restart systemd-networkd

# Verify routes
ip route show

# Check specific route
ip route show 192.168.50.0/24

# Show routes for eth0
networkctl status eth0
```

## Conclusion

Static routes in systemd-networkd are defined in `[Route]` sections within `.network` files. Each `[Route]` section takes `Destination`, `Gateway`, `Metric`, `Type`, and other parameters. Routes follow the interface lifecycle — applied when up, removed when down. Verify with `ip route show` after applying.
