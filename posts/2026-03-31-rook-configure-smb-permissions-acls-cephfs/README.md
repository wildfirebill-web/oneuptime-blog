# How to Configure SMB Permissions and ACLs with CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, SMB, ACL, Permission

Description: Learn how to configure SMB share permissions and POSIX/Windows ACLs on CephFS-backed Samba shares for fine-grained access control.

---

## Permission Models for CephFS SMB

CephFS supports two ACL models when accessed via Samba:

1. **POSIX ACLs** - Standard Unix permissions extended with `setfacl/getfacl` for additional users and groups
2. **Windows ACLs** - Full NT ACL semantics stored in CephFS extended attributes using the `acl_xattr` VFS module

## Enabling ACL Support in smb.conf

For full Windows ACL support, enable the `acl_xattr` VFS module:

```ini
[cephshare]
    vfs objects = ceph acl_xattr
    ceph:config_file = /etc/ceph/ceph.conf
    ceph:user_id = samba
    map acl inherit = yes
    store dos attributes = yes
    kernel share modes = no
    inherit acls = yes
```

## Setting POSIX ACLs on CephFS

Apply extended POSIX ACLs to control access at a granular level:

```bash
# Allow engineering group read/write access to a directory
setfacl -m g:engineering-team:rwX /mnt/cephfs/volumes/smb-shares/engineering

# Allow specific user read-only access
setfacl -m u:contractor1:r-X /mnt/cephfs/volumes/smb-shares/engineering

# Set default ACL so new files inherit parent permissions
setfacl -d -m g:engineering-team:rwX /mnt/cephfs/volumes/smb-shares/engineering
setfacl -d -m u:contractor1:r-X /mnt/cephfs/volumes/smb-shares/engineering

# Verify ACLs
getfacl /mnt/cephfs/volumes/smb-shares/engineering
```

## Setting Windows ACLs via PowerShell

When `acl_xattr` is enabled, Windows ACLs can be managed from Windows clients:

```powershell
# Get current ACL
$acl = Get-Acl "Z:\engineering"

# Create a new access rule
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "EXAMPLE\contractor1",
    "ReadAndExecute",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)

# Add the rule and apply
$acl.AddAccessRule($rule)
Set-Acl "Z:\engineering" $acl
```

## Removing Inherited Permissions

To prevent permission inheritance from the parent:

```powershell
$acl = Get-Acl "Z:\engineering\confidential"
$acl.SetAccessRuleProtection($true, $false)  # Disable inheritance, remove inherited rules
$acl | Set-Acl "Z:\engineering\confidential"
```

## Configuring Share-Level Permissions

In addition to file-level ACLs, set share-level access in `smb.conf`:

```ini
[cephshare]
    valid users = @engineering-team @managers
    read list = @contractors
    write list = @engineering-team
    admin users = smbadmin
    hosts allow = 10.0.0.0/24 192.168.1.0/24
    hosts deny = 0.0.0.0/0
```

Share-level and file-level permissions are both enforced - the most restrictive permission wins.

## Auditing File Access

Enable auditing to track who accesses files:

```ini
[cephshare]
    full_audit:prefix = %u|%I|%m
    full_audit:success = open read write rename mkdir rmdir
    full_audit:failure = connect
    full_audit:facility = LOCAL7
    full_audit:priority = NOTICE
    vfs objects = ceph acl_xattr full_audit
```

Configure syslog to capture the audit logs:

```bash
echo "local7.notice /var/log/samba-audit.log" >> /etc/rsyslog.conf
systemctl restart rsyslog
```

## Summary

CephFS SMB permissions can be managed through POSIX ACLs set at the filesystem level or Windows ACLs stored in extended attributes via the `acl_xattr` Samba module. Combining share-level access restrictions in `smb.conf` with file-level ACLs provides defense-in-depth access control, while audit logging ensures all access is tracked for compliance and security monitoring.
