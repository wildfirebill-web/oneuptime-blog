# How to Troubleshoot DHCP Lease Renewal Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Troubleshooting, Networking, Lease Renewal, Sysadmin

Description: DHCP lease renewal failures occur when the client cannot reach the server at T1 or T2, leading to address expiry and network loss, resolved by diagnosing connectivity, firewall rules, and server...

## Understanding the Renewal Timeline

```text
T=0:    Lease obtained (DHCPACK)
T=50%:  T1 - Client sends DHCPREQUEST unicast to original server
T=87.5%: T2 - Client broadcasts DHCPREQUEST to any server
T=100%:  Lease expires - client loses IP address
```

If renewal fails at T1, the client tries at T2. If T2 also fails, the IP is lost.

## Common Causes

1. DHCP server is down or unreachable
2. Firewall blocking UDP 67/68
3. Network change (VLAN assignment changed)
4. DHCP server ran out of addresses
5. Server rejected the renewal (MAC changed, policy change)

## Diagnosing from the Client Side (Linux)

```bash
# Attempt renewal manually and view verbose output

sudo dhclient -v eth0

# Check current lease expiry
cat /var/lib/dhcp/dhclient.leases | grep -E "expire|renew|rebind"

# Capture the renewal exchange
sudo tcpdump -i eth0 'port 67 or port 68' -w /tmp/renewal.pcap &
sudo dhclient -v eth0
# Stop tcpdump and analyze
```

## Diagnosing from the Server Side

```bash
# ISC dhcpd logs
journalctl -u isc-dhcp-server -f

# Check if server is receiving renewal requests
sudo tcpdump -i eth0 'port 67' -n

# Renewal DHCPREQUEST has:
# - src = client current IP (unicast at T1)
# - option 50 = requested IP
# - option 54 = server identifier
```

## Verifying Network Path to Server

```bash
# Can the client reach the DHCP server?
ping 10.0.0.53        # DHCP server IP

# Test UDP 67 connectivity
nc -u 10.0.0.53 67    # Should not be blocked by firewall

# Check route to server
ip route get 10.0.0.53
```

## Server Log Analysis

```bash
# Look for DHCPREQUEST and DHCPNAK messages
journalctl -u isc-dhcp-server | grep -E "DHCPREQUEST|DHCPACK|DHCPNAK"

# DHCPNAK (Negative Acknowledgment) causes the client to restart with a full DORA:
# DHCPNAK on 192.168.1.105 to 192.168.1.105: client on unknown network
```

## When DHCPNAK Is Returned

The server sends DHCPNAK when:
- The requested IP is no longer valid for the client's network
- The IP has been assigned to another device (conflict)
- A policy change has made the lease invalid

## Key Takeaways

- T1 (50%) failure leads to T2 (87.5%) broadcast; if both fail, the IP expires.
- `dhclient -v eth0` shows the complete renewal exchange with error details.
- DHCPNAK from the server triggers a full DORA cycle.
- Always check both the client's network connectivity and the server's log for DHCPNAK messages.
