# How to Delete a Static Route from the Linux Routing Table

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Networking, Routing, ip command, IPv4, Network Configuration

Description: Remove a specific static route from the Linux routing table using ip route del, handle route deletion errors, and clean up persistent route configuration.

## Introduction

Removing an incorrect or obsolete static route restores normal routing behavior. The `ip route del` command removes a specific route from the kernel's routing table immediately.

## Deleting a Route by Destination

The simplest form — specify only the destination network:

```bash
# Delete the route to 10.10.0.0/16
sudo ip route del 10.10.0.0/16

# Verify removal
ip route show 10.10.0.0/16
# Should return no output
```

## Deleting a Route with a Specific Gateway

If multiple routes exist for the same destination (ECMP or different metrics), specify the gateway to delete only the intended one:

```bash
# Delete the route to 10.10.0.0/16 specifically via 192.168.1.254
sudo ip route del 10.10.0.0/16 via 192.168.1.254
```

## Deleting a Route on a Specific Interface

```bash
# Delete the route to 10.10.0.0/16 via eth1 specifically
sudo ip route del 10.10.0.0/16 dev eth1
```

## Deleting the Default Route

```bash
# Delete the default route (0.0.0.0/0)
sudo ip route del default

# Or specify the gateway to be safe
sudo ip route del default via 192.168.1.1
```

**Warning:** Deleting the default route immediately breaks all connectivity to off-subnet hosts.

## Handling "No such process" Error

This error means the route does not exist in the routing table:

```bash
sudo ip route del 10.10.0.0/16
# RTNETLINK answers: No such process
```

Check if the route exists first:

```bash
ip route show | grep "10.10.0.0"
```

If not found, the route may have already been deleted or was never added.

## Conditional Delete Script

```bash
#!/bin/bash
# Only delete the route if it exists

ROUTE="10.10.0.0/16"
if ip route show | grep -q "$ROUTE"; then
    sudo ip route del "$ROUTE"
    echo "Route $ROUTE deleted"
else
    echo "Route $ROUTE not found — nothing to delete"
fi
```

## Removing Persistent Route Configuration

After deleting the runtime route, remove it from persistent configuration to prevent it from coming back on reboot:

**Netplan:**

```bash
sudo nano /etc/netplan/01-netcfg.yaml
# Remove the route entry from the routes: list
sudo netplan apply
```

**NetworkManager:**

```bash
nmcli con mod "Wired connection 1" -ipv4.routes "10.10.0.0/16 192.168.1.254"
nmcli con up "Wired connection 1"
```

**Debian /etc/network/interfaces:**

```bash
sudo nano /etc/network/interfaces
# Remove or comment out the post-up ip route add line
sudo systemctl restart networking
```

## Flushing All Routes (Caution)

```bash
# Remove ALL routes (will break connectivity immediately!)
sudo ip route flush table main
```

Only use this in a recovery scenario or when rebuilding routing from scratch.

## Conclusion

`ip route del <destination>/<prefix>` removes a route immediately. Specify the gateway when multiple paths exist for the same destination. Always clean up the persistent configuration to prevent the route from returning on the next reboot.
