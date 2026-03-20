# How to Release a DHCP Lease

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Networking, Linux, Windows, macOS, Network Diagnostics

Description: Releasing a DHCP lease sends a DHCPRELEASE message to the server, allowing it to return the IP to the available pool immediately rather than waiting for the lease to expire.

## What Happens During Release

1. Client sends a **DHCPRELEASE** message (unicast) to the DHCP server.
2. Server marks the IP as available in its lease pool.
3. Client removes the IP from its interface.
4. The interface is left without an IP until a new lease is obtained.

## Linux: dhclient

```bash
# Release the lease for eth0

sudo dhclient -r eth0

# Verify the IP is removed
ip addr show eth0
# The IP should no longer be listed

# Release all leases (all interfaces)
sudo dhclient -r
```

## Linux: NetworkManager

```bash
# Disconnect the connection (releases lease)
nmcli connection down "Wired connection 1"

# Remove the IP without fully disconnecting
nmcli device modify eth0 ipv4.method disabled
```

## Windows: ipconfig

```cmd
REM Release lease for all adapters
ipconfig /release

REM Release lease for a specific adapter
ipconfig /release "Ethernet"
ipconfig /release "Wi-Fi"

REM Verify IP is removed
ipconfig
REM Should show "Media disconnected" or no IP
```

## macOS: ipconfig

```bash
# Release the lease
sudo ipconfig set en0 NONE

# Verify
ifconfig en0 | grep inet
# Should show no inet address
```

## When to Release a Lease

- Before shutting down a server that holds a reserved IP.
- When moving to a different VLAN or network.
- When testing DHCP pool exhaustion behavior.
- When diagnosing IP conflict issues.
- Before changing from DHCP to static IP.

## DHCPRELEASE Message Format

```python
# Simulating what dhclient sends (conceptual)
# DHCPRELEASE is a DHCPREQUEST with DHCP Message Type option = 7
# Sent as unicast directly to the DHCP server (not broadcast)

# The server receives it and marks the IP as available:
# Lease 192.168.1.105: status -> AVAILABLE
```

## Key Takeaways

- Releasing a lease returns the IP to the DHCP pool immediately.
- On Linux: `dhclient -r eth0`; Windows: `ipconfig /release`; macOS: `ipconfig set en0 NONE`.
- A DHCPRELEASE is sent unicast to the DHCP server - not broadcast.
- If the server is unreachable, the client still removes its IP locally, and the server will time out the lease naturally.
