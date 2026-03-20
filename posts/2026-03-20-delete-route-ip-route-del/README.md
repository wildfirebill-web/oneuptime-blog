# How to Delete a Route with ip route del

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Routing, Linux, Networking, Ip-command, IPv4, Troubleshooting

Description: Remove unwanted or incorrect IPv4 routes from the Linux kernel routing table using the ip route del command.

## Introduction

Incorrect routes can cause traffic to be sent to the wrong gateway or to fail entirely. The `ip route del` command removes specific routes from the Linux routing table, giving you precise control over traffic paths without disrupting other routes.

## Viewing Routes Before Deletion

Always verify the exact route before deleting:

```bash
# Display the full routing table

ip route show

# Find a specific route
ip route show | grep 10.50.0.0
```

## Deleting a Default Route

```bash
# Remove the default route (use with care - disconnects all non-local traffic)
sudo ip route del default

# Delete default route via a specific gateway
sudo ip route del default via 192.168.1.1
```

## Deleting a Specific Network Route

```bash
# Delete the route to 10.50.0.0/16
sudo ip route del 10.50.0.0/16

# Delete route via a specific gateway (when multiple routes exist for the same prefix)
sudo ip route del 10.50.0.0/16 via 192.168.1.10
```

## Deleting Routes by Interface

When you want to remove all routes associated with an interface:

```bash
# Flush all routes on a specific interface
sudo ip route flush dev tun0
```

## Deleting a Host Route

```bash
# Remove a /32 host route
sudo ip route del 8.8.8.8/32 via 10.0.0.1
```

## Removing Blackhole and Unreachable Routes

```bash
# Remove a blackhole route
sudo ip route del blackhole 192.0.2.0/24
```

## Verifying Deletion

```bash
# Confirm the route is gone
ip route show | grep 10.50.0.0

# Check which route would now be used
ip route get 10.50.0.1
```

## Handling "No such process" Errors

If you see `RTNETLINK answers: No such process`, the route does not exist exactly as specified. Try:

```bash
# Show route details to find the exact specification
ip route show 10.50.0.0/16

# Include via and dev in the delete command to match exactly
sudo ip route del 10.50.0.0/16 via 192.168.1.10 dev eth0
```

## Replacing a Route

Instead of deleting and re-adding, use `ip route replace`:

```bash
# Replace existing route for 10.50.0.0/16 with a new gateway
sudo ip route replace 10.50.0.0/16 via 192.168.1.20
```

## Persistence

Like `ip route add`, deletions made with `ip route del` do not survive reboots. Update your persistent network configuration (Netplan, NetworkManager, or `/etc/network/interfaces`) to reflect the desired routing state.

## Conclusion

`ip route del` is essential for cleaning up incorrect or outdated routes. Always combine it with `ip route get` to verify the new routing decision after deletion.
