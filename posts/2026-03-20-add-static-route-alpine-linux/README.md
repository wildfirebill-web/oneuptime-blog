# How to Add a Static Route on Alpine Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Alpine Linux, Static Routes, Networking, Routing, /etc/network/interfaces

Description: Add persistent static routes on Alpine Linux using the ip route command and /etc/network/interfaces post-up directives or the Alpine-specific iproute2 configuration.

## Introduction

Alpine Linux uses a minimal network configuration stack based on `/etc/network/interfaces` with BusyBox `ip` commands for route management. Static routes are added temporarily with `ip route add` or persistently via `post-up` directives in the interfaces file.

## Add a Temporary Route

```bash
# Add a static route using ip route
ip route add 192.168.2.0/24 via 10.0.0.1

# Add using BusyBox route command (legacy)
route add -net 192.168.2.0/24 gw 10.0.0.1

# View routing table
ip route show
```

## Persistent Routes via /etc/network/interfaces

```bash
# /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 10.0.0.100
    netmask 255.255.255.0
    gateway 10.0.0.1
    # Add persistent static routes
    post-up ip route add 192.168.2.0/24 via 10.0.0.1
    post-up ip route add 172.16.0.0/16 via 10.0.0.1
    pre-down ip route del 192.168.2.0/24 via 10.0.0.1
    pre-down ip route del 172.16.0.0/16 via 10.0.0.1
```

## Apply the Configuration

```bash
# Restart networking to apply
/etc/init.d/networking restart

# Or use ifupdown
ifdown eth0 && ifup eth0

# Verify
ip route show
```

## Routes with OpenRC Service

Create a startup script for routes:

```bash
cat > /etc/local.d/static-routes.start << 'EOF'
#!/bin/sh
ip route add 192.168.2.0/24 via 10.0.0.1 2>/dev/null
ip route add 172.16.0.0/16 via 10.0.0.1 2>/dev/null
EOF
chmod +x /etc/local.d/static-routes.start

# Enable local service
rc-update add local default
```

## Alpine with DHCP + Extra Routes

```bash
auto eth0
iface eth0 inet dhcp
    post-up ip route add 192.168.2.0/24 via $(ip route show default | awk '{print $3}')
```

## Docker/Container Alpine

In Alpine containers, routes are often added via environment variables or startup scripts since there is no init system:

```dockerfile
# Dockerfile
FROM alpine:latest
RUN apk add --no-cache iproute2
CMD ["/bin/sh", "-c", "ip route add 192.168.2.0/24 via 10.0.0.1; exec myapp"]
```

## Conclusion

Alpine Linux static routes use `ip route add` for temporary configuration and `post-up` directives in `/etc/network/interfaces` for persistence. The minimal Alpine environment means no NetworkManager or Netplan — stick with the interfaces file or OpenRC local scripts for persistent routing. The `/etc/local.d/*.start` pattern integrates cleanly with Alpine's OpenRC init system.
