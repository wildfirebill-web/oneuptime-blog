# How to Troubleshoot Client Connection Issues to Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Client, Network, Storage

Description: Diagnose and fix Ceph client connection failures including monitor connection errors, auth failures, and OSD connection timeouts.

---

Client connection issues are among the most common Ceph problems. They range from simple misconfiguration to network problems and authentication failures. This guide walks through a systematic diagnostic process.

## Step 1 - Verify Basic Connectivity

Before debugging Ceph-specific issues, confirm the client can reach the monitors:

```bash
# Test TCP connectivity to monitor ports
nc -zv 192.168.1.10 6789
nc -zv 192.168.1.10 3300

# Ping monitors
ping -c 3 192.168.1.10

# Check route to monitor network
traceroute 192.168.1.10
```

## Step 2 - Check ceph.conf

Verify the client's `ceph.conf` has the correct monitor addresses:

```bash
cat /etc/ceph/ceph.conf
ceph -s --debug-ms 1  # show connection attempts
```

Common misconfigurations:
- Wrong monitor IP addresses
- Wrong FSID
- Auth settings set to `none` when cluster requires `cephx`

## Step 3 - Diagnose Authentication Errors

```bash
# Test with explicit keyring
ceph --user admin --keyring /etc/ceph/ceph.client.admin.keyring status

# Check auth error in monitor log
journalctl -u ceph-mon@mon1 | grep -i "auth\|error\|denied"
```

If you see `auth: error, denied`, the keyring is wrong or the user does not exist:

```bash
# Verify the key matches
ceph auth get client.myapp
cat /etc/ceph/ceph.client.myapp.keyring
```

## Step 4 - Check Firewall Rules

```bash
# Check iptables/nftables on the client
iptables -L -n | grep -E "6789|3300|6800"

# Check firewall on monitor and OSD nodes
firewall-cmd --list-all

# Required open ports
# Monitors: 6789, 3300
# OSDs: 6800-7300
```

## Step 5 - Debug Messenger Layer

Increase Ceph debug logging on the client to see exactly what is happening:

```bash
ceph --debug-ms 5 --debug-auth 5 -s 2>&1 | head -100
```

Common error messages and meanings:

| Error | Cause |
|-------|-------|
| `ECONNREFUSED` | Monitor port not reachable |
| `auth: error` | Wrong key or user does not exist |
| `wrong fsid` | ceph.conf has wrong cluster FSID |
| `crush: wrong epoch` | Client has stale CRUSH map |

## Step 6 - Check OSD Connectivity

If monitor connection works but OSD operations fail:

```bash
# Test OSD port range
for port in $(seq 6800 6810); do
  nc -zv 192.168.1.20 $port 2>&1 | grep -v "refused"
done
```

## Step 7 - Check Client Network Interface

```bash
# Verify the client sends traffic from the correct interface
ip route get 192.168.1.10

# If using a dedicated storage interface, ensure it is up
ip addr show eth1
```

Configure a specific interface in `ceph.conf`:

```ini
[client]
public_addr = 192.168.1.50
```

## Enabling Persistent Debug Logs

```ini
[client]
log_file = /var/log/ceph/ceph-client.log
debug_ms = 1
debug_auth = 1
```

## Summary

Troubleshooting Ceph client connections follows a layered approach: first verify basic TCP reachability to monitor ports, then check authentication credentials, then inspect firewall rules for both monitor and OSD port ranges. Using `--debug-ms 5` on the client side reveals detailed connection negotiation that quickly identifies the root cause of most failures.
