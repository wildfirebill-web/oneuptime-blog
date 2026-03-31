# How to Troubleshoot SMB Sharing Issues with CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, SMB, Troubleshooting, CephFS

Description: Diagnose and fix common SMB sharing issues with CephFS-backed Samba including authentication failures, permission errors, and connectivity problems.

---

## Common SMB Issues with CephFS

The most frequent issues when sharing CephFS via Samba fall into these categories:
- Authentication and login failures
- Permission denied errors on files or directories
- CephFS mount or VFS module errors
- Slow performance or disconnects
- Share not visible to clients

## Diagnosing Authentication Failures

If clients cannot authenticate, check Samba logs first:

```bash
tail -100 /var/log/samba/log.smbd
grep -i "authentication\|password\|denied" /var/log/samba/log.smbd | tail -20
```

Verify that the Samba user exists and has a valid password:

```bash
pdbedit -L | grep alice
smbpasswd -e alice  # Enable account if disabled
```

Test authentication from the command line:

```bash
smbclient //localhost/cephshare -U alice%Password123 -c "ls"
```

For AD-joined setups, check winbind status:

```bash
wbinfo --ping-dc
wbinfo -a EXAMPLE\\alice%Password123
systemctl status winbind
```

## Diagnosing Permission Denied Errors

If authentication succeeds but file operations fail, check POSIX permissions:

```bash
# Check permissions on the share root
ls -la /mnt/cephfs/volumes/smb-shares/share1

# Check ACLs
getfacl /mnt/cephfs/volumes/smb-shares/share1
```

Verify the Ceph client user has access to the path:

```bash
ceph auth get client.samba
# Check mds capability includes the share path
```

Check that the Samba process has the correct Linux user:

```bash
ps aux | grep smbd
# Should run as root for impersonation
```

## Diagnosing CephFS VFS Module Errors

Check Samba logs for VFS initialization errors:

```bash
grep -i "ceph\|vfs\|libceph" /var/log/samba/log.smbd | tail -30
```

Common VFS errors and fixes:

```bash
# "Failed to open ceph.conf" - Check file exists and is readable
ls -la /etc/ceph/ceph.conf

# "ceph: authentication error" - Check keyring
ls -la /etc/ceph/ceph.client.samba.keyring
ceph auth get client.samba

# "ceph: MDS is not available" - Check MDS health
ceph fs status
ceph mds stat
```

## Testing with testparm

Verify the smb.conf configuration:

```bash
testparm -s
testparm --parameter-name="vfs objects" --section-name="cephshare"
```

## Diagnosing Slow Performance

Enable timing information in Samba logs:

```bash
# Set log level temporarily
smbcontrol smbd debug "5"
```

Check for CephFS MDS latency:

```bash
ceph daemon mds.0 perf dump | python3 -m json.tool | grep latency
```

Check CephFS slow requests:

```bash
ceph daemon mds.0 dump_ops_in_flight
```

## Fixing Share Visibility Issues

If a share is not visible to clients:

```bash
# Verify share is listed
smbclient -L //localhost -N

# Check browsing is enabled in smb.conf
grep "browsable" /etc/samba/smb.conf

# Reload Samba config
smbcontrol smbd reload-config
```

## Re-testing After Changes

After making configuration changes, restart services and re-test:

```bash
systemctl restart smb winbind
smbclient //localhost/cephshare -U testuser%password -c "ls; mkdir testdir; rmdir testdir"
```

## Summary

Troubleshooting CephFS SMB issues requires checking multiple layers starting with Samba authentication logs, then POSIX permissions and ACLs, then the CephFS VFS module configuration. The majority of issues are caused by misconfigured POSIX permissions, missing Ceph keyring capabilities, or incorrect winbind configuration for AD environments. Using `smbclient` locally for testing isolates issues quickly before involving clients.
