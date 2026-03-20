# How to Configure vsftpd to Listen on IPv4 Only

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: vsftpd, FTP, IPv4, Configuration, File Transfer, Server

Description: Configure vsftpd to listen exclusively on IPv4 interfaces by setting listen=YES and listen_ipv6=NO, and optionally binding to a specific IPv4 address.

## Introduction

vsftpd defaults vary by distribution-on some it listens on IPv6 (which may or may not also accept IPv4 depending on system config). Setting `listen=YES` and `listen_ipv6=NO` ensures IPv4-only operation.

## Basic IPv4-Only Configuration

```bash
# /etc/vsftpd.conf

# Listen on IPv4 (standalone mode)

listen=YES

# Disable IPv6 listening
listen_ipv6=NO

# Anonymous access disabled (security best practice)
anonymous_enable=NO

# Local users can log in
local_enable=YES

# Allow write operations
write_enable=YES

# Use local umask for new files
local_umask=022

# Log all FTP requests and responses
xferlog_enable=YES
xferlog_std_format=YES

# Enable ASCII upload/download
ascii_upload_enable=YES
ascii_download_enable=YES
```

## Bind to Specific IPv4 Address

```bash
# /etc/vsftpd.conf

listen=YES
listen_ipv6=NO

# Bind vsftpd to a specific IPv4 address
listen_address=203.0.113.10
```

## Verifying IPv4-Only Listening

```bash
# Restart vsftpd
sudo systemctl restart vsftpd

# Check listening sockets
sudo ss -tlnp | grep vsftpd
# Expected: LISTEN 0 32 0.0.0.0:21 (or 203.0.113.10:21)
# NOT: LISTEN 0 32 [::]:21

# Test connection
ftp 203.0.113.10
# Or
curl -v ftp://203.0.113.10/

# Check vsftpd log
sudo tail -f /var/log/vsftpd.log
```

## Complete Secure vsftpd Configuration

```bash
# /etc/vsftpd.conf

# IPv4 only
listen=YES
listen_ipv6=NO
listen_address=203.0.113.10

# Authentication
anonymous_enable=NO
local_enable=YES
write_enable=YES

# Chroot users to their home directories
chroot_local_user=YES
allow_writeable_chroot=YES

# Passive mode (required for firewall compatibility)
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000
pasv_address=203.0.113.10   # Public IP for passive mode

# Security
hide_ids=YES
ls_recurse_enable=NO
chmod_enable=YES

# Logging
xferlog_enable=YES
log_ftp_protocol=YES
vsftpd_log_file=/var/log/vsftpd.log
```

## Conclusion

Setting `listen=YES` and `listen_ipv6=NO` in `/etc/vsftpd.conf` forces vsftpd to use only IPv4. Use `listen_address` to bind to a specific IPv4 rather than all interfaces. Verify with `ss -tlnp` that only IPv4 sockets appear after restart. Always enable passive mode (`pasv_enable=YES`) with a defined port range for clients behind NAT to work correctly.
