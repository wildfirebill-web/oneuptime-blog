# How to Delete a Static Route on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Routing, Linux, IPv4

Description: Learn how to delete static routes on Linux using ip route del and how to remove persistent routes from various configuration files.

## Deleting a Route with `ip route del`

```bash
# Delete a specific network route
ip route del 10.0.0.0/24

# Delete route via a specific gateway
ip route del 10.0.0.0/24 via 192.168.1.254

# Delete route on a specific interface
ip route del 10.0.0.0/24 dev eth0

# Delete the default gateway
ip route del default

# Delete default via specific gateway
ip route del default via 192.168.1.1
```

## Verifying Deletion

```bash
# Check the route is removed
ip route show 10.0.0.0/24
# Should return no output if successfully deleted

# Or check full table
ip route show
```

## Deleting Multiple Routes

```bash
#!/bin/bash
# Delete multiple routes
ROUTES_TO_DELETE=(
    "10.10.0.0/24"
    "10.20.0.0/24"
    "172.16.5.0/24"
)

for route in "${ROUTES_TO_DELETE[@]}"; do
    if ip route del "$route" 2>/dev/null; then
        echo "Deleted: $route"
    else
        echo "Route not found: $route"
    fi
done
```

## Flush All Routes for an Interface

```bash
# Remove all routes on eth0
ip route flush dev eth0

# Flush all routes in the main table (dangerous!)
ip route flush table main
```

## Removing Persistent Routes

### Netplan (Ubuntu/Debian)

```bash
# Edit /etc/netplan/01-config.yaml
# Remove the route entries under 'routes:'
# Then apply:
sudo netplan apply
```

### RHEL/CentOS 7 (ifcfg)

```bash
# Edit /etc/sysconfig/network-scripts/route-eth0
# Remove the relevant lines
# Restart network:
systemctl restart network
```

### NetworkManager (RHEL 8+)

```bash
# Remove route from connection
nmcli connection modify eth0 -ipv4.routes "10.0.0.0/24 192.168.1.254"

# Apply changes
nmcli connection up eth0

# Verify
nmcli connection show eth0 | grep route
```

### systemd-networkd

```bash
# Edit /etc/systemd/network/10-eth0.network
# Remove the [Route] sections
# Restart systemd-networkd:
systemctl restart systemd-networkd
```

## Error: Route Not Found

```bash
ip route del 10.0.0.0/24 via 192.168.1.254
# Error: RTNETLINK answers: No such process
```

This means the route doesn't match exactly. Check:

```bash
# Show exact route details
ip route show 10.0.0.0/24

# Then delete with the exact parameters shown
ip route del 10.0.0.0/24 dev eth0 proto static
```

## Legacy `route` Command

```bash
# Delete route using legacy command
route del -net 10.0.0.0 netmask 255.255.255.0 gw 192.168.1.254

# Delete default gateway
route del default gw 192.168.1.1
```

## Key Takeaways

- `ip route del NETWORK` removes a route by destination prefix.
- Include `via GATEWAY` or `dev INTERFACE` if multiple routes to the same destination exist.
- `ip route flush dev IFACE` removes all routes for that interface.
- For persistent routes, also update the appropriate network configuration file.

**Related Reading:**

- [How to Add a Static Route on Linux](https://oneuptime.com/blog/post/2026-03-20-add-static-route-linux/view)
- [How to View the Routing Table on Linux](https://oneuptime.com/blog/post/2026-03-20-view-routing-table-linux/view)
- [How to Configure a Default Gateway on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-default-gateway-linux/view)
