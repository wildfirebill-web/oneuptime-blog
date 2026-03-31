# How to Troubleshoot iSCSI Connection Issues with Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, iSCSI, Troubleshooting, Debugging

Description: Diagnose and fix common iSCSI connection issues with Ceph gateways including discovery failures, login errors, and dropped sessions.

---

## Common iSCSI Connection Issues

The most frequent iSCSI connection problems with Ceph gateways are:
- Target discovery failures
- Login authentication errors
- Session drops under load
- Path not available after gateway restart

## Step 1 - Verify Network Connectivity

Before debugging iSCSI-specific issues, confirm basic network connectivity to the gateway:

```bash
ping -c 5 10.0.1.10
nc -zv 10.0.1.10 3260
```

If port 3260 is not reachable, check firewall rules:

```bash
# On the gateway
firewall-cmd --list-ports | grep 3260
firewall-cmd --add-port=3260/tcp --permanent
firewall-cmd --reload
```

Or with iptables:

```bash
iptables -I INPUT -p tcp --dport 3260 -j ACCEPT
```

## Step 2 - Debug Discovery Failures

If `iscsiadm` discovery returns no targets, run with debug output:

```bash
iscsiadm -m discovery -t sendtargets -p 10.0.1.10:3260 -d8 2>&1 | tail -40
```

On the gateway, check whether the target service is running:

```bash
systemctl status rbd-target-gw
targetcli ls /iscsi
```

Check for LIO kernel module issues:

```bash
lsmod | grep -E "target|iscsi"
dmesg | grep -i iscsi | tail -20
```

## Step 3 - Diagnose Login Errors

If discovery succeeds but login fails, check the error messages:

```bash
iscsiadm -m node -T iqn.2024-01.com.example:storage -p 10.0.1.10 --login -d8 2>&1
```

Common errors and causes:

- `CHAP authentication error` - Mismatched username or password
- `initiator name is not authorized` - ACL not configured for this initiator
- `connection refused` - Gateway service not running

Check ACL configuration on the gateway:

```bash
gwcli
/iscsi-targets/iqn.../hosts> ls
```

Ensure the initiator IQN is listed. If not, add it:

```bash
/hosts> create iqn.1993-08.org.debian:client01
/hosts/iqn.1993-08.org.debian:client01/> cd luns
/luns> add rbd/iscsi/vol1
```

## Step 4 - Fix Session Drops

iSCSI sessions may drop due to network timeouts or gateway overload. Increase timeout values:

```bash
iscsiadm -m node -T iqn.2024-01.com.example:storage \
  --op update -n node.conn[0].timeo.noop_out_interval -v 5
iscsiadm -m node -T iqn.2024-01.com.example:storage \
  --op update -n node.conn[0].timeo.noop_out_timeout -v 30
iscsiadm -m node -T iqn.2024-01.com.example:storage \
  --op update -n node.session.timeo.replacement_timeout -v 120
```

Check for errors in the kernel log:

```bash
dmesg -T | grep -i iscsi | grep -E "error|failed|timeout" | tail -20
```

## Step 5 - Checking Gateway Logs

Inspect gateway logs for errors:

```bash
journalctl -u rbd-target-gw -n 100 --no-pager
journalctl -u rbd-target-api -n 100 --no-pager
```

Enable debug logging for the gateway:

```bash
cat >> /etc/ceph/iscsi-gateway.cfg << 'EOF'
log_level = debug
EOF
systemctl restart rbd-target-gw
```

## Step 6 - Reinitializing a Failed Session

If a session is stuck in a failed state, log out and reconnect:

```bash
iscsiadm -m node -T iqn.2024-01.com.example:storage --logout
iscsiadm -m node -T iqn.2024-01.com.example:storage --login
```

## Summary

Troubleshooting Ceph iSCSI connection issues requires a layered approach starting with network reachability, progressing through target discovery and login authentication, and finally investigating session stability. Debug flags in `iscsiadm` and `journalctl` logs from the gateway services provide the detailed information needed to isolate and fix most connection problems.
