# How to Use bind interfaces only in Samba for IPv4-Only Listening

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Samba, IPv4, Bind interfaces only, SMB, Security, Network Isolation

Description: Configure Samba's bind interfaces only directive to ensure it only listens on specified IPv4 interfaces, preventing unintended exposure on other network interfaces or IPv6.

## Introduction

The `bind interfaces only = yes` directive in Samba ensures that smbd and nmbd only create sockets on the interfaces listed in the `interfaces` parameter. Without it, Samba listens on all interfaces regardless of the `interfaces` setting.

## How bind interfaces only Works

```bash
# Without bind interfaces only:

# interfaces = eth0 lo
# Samba still listens on ALL interfaces for some operations
# (only restricts certain lookups, not all binding)

# With bind interfaces only = yes:
# Samba ONLY creates listening sockets on eth0 and lo
# All other interfaces (eth1, docker0, etc.) are completely ignored
```

## Configuration

```bash
# /etc/samba/smb.conf

[global]
   # List only the interfaces Samba should use
   interfaces = lo 10.0.0.5
   bind interfaces only = yes

   workgroup = WORKGROUP
   server string = Samba Server
   security = user

   # Disable IPv6 completely
   dns proxy = no
```

## Multi-Interface Server Example

```bash
# Server has:
# eth0: 10.0.0.5    (internal LAN - Samba should be here)
# eth1: 203.0.113.10 (public internet - Samba should NOT be here)
# docker0: 172.17.0.1 (Docker bridge - Samba should NOT be here)

# /etc/samba/smb.conf
[global]
   # Only internal LAN and loopback
   interfaces = eth0 lo
   bind interfaces only = yes

   # This prevents Samba from being accessible on eth1 and docker0
```

## Verifying the Configuration

```bash
# Verify with testparm
sudo testparm -s | grep -E "interfaces|bind interfaces"

# Check what ports Samba is actually listening on
sudo ss -tlnp | grep smbd
# Expected: only 10.0.0.5:445 and 127.0.0.1:445
# NOT: 203.0.113.10:445 or 0.0.0.0:445

# Verify nmbd binding (NetBIOS)
sudo ss -ulnp | grep nmbd

# Test that external IP is NOT accessible
smbclient -L //203.0.113.10    # Should time out or connection refused
smbclient -L //10.0.0.5        # Should show shares
```

## Combining with hosts allow for Defense in Depth

```bash
# /etc/samba/smb.conf

[global]
   # Layer 1: Bind only to internal interface
   interfaces = 10.0.0.5 lo
   bind interfaces only = yes

   # Layer 2: Only allow connections from these IPs
   hosts allow = 10.0.0.0/24 127.0.0.1
   hosts deny = ALL

   workgroup = WORKGROUP
   security = user
```

## Reloading After Changes

```bash
# Reload Samba configuration
sudo systemctl reload smbd
sudo systemctl reload nmbd

# Or restart:
sudo systemctl restart smbd nmbd

# Check status
sudo systemctl status smbd
sudo smbstatus --version

# Confirm no errors in log
sudo tail -20 /var/log/samba/log.smbd
```

## Conclusion

`bind interfaces only = yes` is the critical parameter that enforces interface binding in Samba. Pair it with an explicit `interfaces` list containing only the IPv4 addresses Samba should use. This prevents Samba from being accidentally exposed on public interfaces, Docker bridges, or IPv6 addresses. Combine with `hosts allow` for additional IP-level access control.
