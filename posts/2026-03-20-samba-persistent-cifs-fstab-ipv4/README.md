# How to Configure Persistent CIFS/Samba Mounts via /etc/fstab on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Samba, CIFS, IPv4, fstab, Persistent Mount, Automount

Description: Add CIFS/Samba mounts to /etc/fstab for automatic mounting at boot, use credential files for secure password storage, and configure systemd automount as an alternative.

## Introduction

Persistent CIFS mounts via `/etc/fstab` mount Samba shares automatically at boot. Key challenges are: storing credentials securely, ensuring network is available before mounting, and preventing boot hangs if the server is unreachable.

## Credentials File Setup

```bash
# Never put passwords directly in /etc/fstab
# Create a credentials file:
sudo cat > /etc/samba/credentials-fileserver << 'EOF'
username=smbuser
password=secretpassword
domain=WORKGROUP
EOF

sudo chmod 600 /etc/samba/credentials-fileserver
sudo chown root:root /etc/samba/credentials-fileserver
```

## /etc/fstab Entry

```bash
# /etc/fstab
# Format: //server/share  /mountpoint  cifs  options  0  0

# Basic persistent mount
//203.0.113.10/shared  /mnt/samba  cifs  credentials=/etc/samba/credentials-fileserver,_netdev  0  0

# With user permissions and SMB version
//203.0.113.10/shared  /mnt/samba  cifs  \
  credentials=/etc/samba/credentials-fileserver,\
  uid=1000,gid=1000,\
  file_mode=0664,dir_mode=0775,\
  vers=3.0,\
  _netdev  0  0
```

## Critical fstab Options

| Option | Purpose |
|---|---|
| `_netdev` | Wait for network before mounting (prevents boot hang) |
| `credentials=` | Secure credential storage |
| `uid=1000` | Map files to local user |
| `gid=1000` | Map files to local group |
| `vers=3.0` | SMB protocol version |
| `soft` | Return error on timeout instead of hanging |
| `noauto` | Don't mount at boot (mount manually) |
| `x-systemd.automount` | Systemd automount |

## Testing fstab Entry

```bash
# Create mount point
sudo mkdir -p /mnt/samba

# Test mount without rebooting
sudo mount -a

# Check if mount succeeded
mount | grep cifs
df -h /mnt/samba

# Test file access
ls /mnt/samba
touch /mnt/samba/fstab-test

# Unmount and remount to verify fstab
sudo umount /mnt/samba
sudo mount /mnt/samba
```

## systemd Automount (On-Demand Mounting)

```bash
# /etc/systemd/system/mnt-samba.mount
cat > /etc/systemd/system/mnt-samba.mount << 'EOF'
[Unit]
Description=CIFS Samba Share
After=network-online.target
Wants=network-online.target

[Mount]
What=//203.0.113.10/shared
Where=/mnt/samba
Type=cifs
Options=credentials=/etc/samba/credentials-fileserver,uid=1000,gid=1000,vers=3.0

[Install]
WantedBy=multi-user.target
EOF

# /etc/systemd/system/mnt-samba.automount
cat > /etc/systemd/system/mnt-samba.automount << 'EOF'
[Unit]
Description=CIFS Samba Automount

[Automount]
Where=/mnt/samba
TimeoutIdleSec=300

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now mnt-samba.automount
```

## Troubleshooting fstab Mount Failures

```bash
# Verbose mount for debugging
sudo mount -v -t cifs //203.0.113.10/shared /mnt/samba \
  -o credentials=/etc/samba/credentials-fileserver

# Check journal for mount errors
sudo journalctl -b | grep -i cifs

# If mount hangs at boot, add x-systemd.device-timeout=30
# in fstab to limit wait time
```

## Conclusion

Add `_netdev` to all CIFS fstab entries so the system waits for the network before mounting. Store credentials in a root-owned file with mode 600. Use `x-systemd.automount` or a systemd automount unit for on-demand mounting that doesn't block boot. Test entries with `mount -a` and verify with `mount | grep cifs` before relying on boot-time mounting.
