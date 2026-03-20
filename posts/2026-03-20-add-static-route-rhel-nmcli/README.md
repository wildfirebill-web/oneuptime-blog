# How to Add a Static Route on RHEL Using nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RHEL, Static Routes, nmcli, NetworkManager, Networking, Routing

Description: Add persistent static routes on Red Hat Enterprise Linux using nmcli to modify NetworkManager connection profiles with route entries.

## Introduction

On RHEL, CentOS, and Fedora, nmcli modifies NetworkManager connection profiles to add persistent static routes. Routes are saved in the connection profile and applied automatically when the connection is active.

## Add a Static Route to a Connection

```bash
# Add a static route to the eth0 connection
nmcli connection modify eth0 \
    +ipv4.routes "192.168.2.0/24 10.0.0.1"

# Apply the change
nmcli connection up eth0

# Verify
ip route show | grep 192.168.2.0
```

## Add Multiple Routes

```bash
# Add multiple routes in one command
nmcli connection modify eth0 \
    +ipv4.routes "192.168.2.0/24 10.0.0.1" \
    +ipv4.routes "172.16.0.0/16 10.0.0.1"
```

## Add a Route with Metric

```bash
nmcli connection modify eth0 \
    +ipv4.routes "192.168.2.0/24 10.0.0.1 100"
# Format: "destination gateway metric"
```

## Add a Host Route

```bash
nmcli connection modify eth0 \
    +ipv4.routes "203.0.113.5/32 10.0.0.1"
```

## View Current Routes in a Connection

```bash
# Show routes in the connection profile
nmcli connection show eth0 | grep ipv4.routes

# Show active routes
ip route show
```

## Remove a Static Route

```bash
# Remove a specific route from the connection
nmcli connection modify eth0 \
    -ipv4.routes "192.168.2.0/24 10.0.0.1"

nmcli connection up eth0
```

## Add Route Using connection.d Files

For advanced route management, create a file in the connection directory:

```bash
# Find the connection UUID
nmcli -g UUID connection show eth0

# Create a route file
cat > /etc/NetworkManager/system-connections/eth0.nmconnection << 'EOF'
[ipv4]
route1=192.168.2.0/24,10.0.0.1,100
route2=172.16.0.0/16,10.0.0.1,200
EOF

nmcli connection reload
nmcli connection up eth0
```

## Conclusion

nmcli adds persistent static routes using `+ipv4.routes "network gateway metric"`. Routes are stored in the connection profile and applied when the connection is active. Use `nmcli connection modify` to add routes and `nmcli connection up` to apply them. Remove routes with `-ipv4.routes` using the same format.
