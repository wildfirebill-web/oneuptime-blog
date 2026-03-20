# How to Use SFTP as a Secure FTP Alternative on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SFTP, SSH, IPv4, Security, File Transfer, OpenSSH

Description: Set up SFTP over SSH as a secure replacement for FTP, configure chroot jails for SFTP-only users, restrict access by IP, and use SFTP with common clients.

## Introduction

SFTP (SSH File Transfer Protocol) runs over SSH on port 22 and provides encrypted file transfers without the complexity of FTP passive mode, NAT traversal, or separate SSL certificates. It uses your existing SSH infrastructure and supports public key authentication.

## SFTP vs FTP Comparison

| Feature | FTP/FTPS | SFTP |
|---|---|---|
| Port | 21 + data ports | 22 only |
| Encryption | Optional (FTPS) | Always encrypted |
| Firewall complexity | High (passive ports) | Low (single port) |
| NAT traversal | Requires configuration | Transparent |
| Authentication | Password or cert | Password or SSH key |
| Protocol | Application layer FTP | SSH subsystem |

## Basic SFTP Setup (Already Included with OpenSSH)

```bash
# SFTP is built into OpenSSH — no additional install needed
# Verify SFTP subsystem is enabled in sshd_config
grep Subsystem /etc/ssh/sshd_config
# Expected: Subsystem sftp /usr/lib/openssh/sftp-server

# Test SFTP connection
sftp user@203.0.113.10
```

## Creating SFTP-Only Users with Chroot

```bash
# /etc/ssh/sshd_config

# SSH server binds to specific IPv4
ListenAddress 203.0.113.10

# SFTP subsystem using internal handler (more efficient)
Subsystem sftp internal-sftp

# Match block for SFTP-only group
Match Group sftpusers
    ChrootDirectory /srv/sftp/%u    # Chroot to per-user directory
    ForceCommand    internal-sftp   # Force SFTP, no shell access
    AllowTcpForwarding no
    X11Forwarding no
    PasswordAuthentication yes
```

```bash
# Create group and user
sudo groupadd sftpusers
sudo useradd -g sftpusers -s /usr/sbin/nologin -M sftpuser1

# Set password
sudo passwd sftpuser1

# Create chroot directory (must be owned by root)
sudo mkdir -p /srv/sftp/sftpuser1
sudo chown root:root /srv/sftp/sftpuser1
sudo chmod 755 /srv/sftp/sftpuser1

# Create upload directory owned by user
sudo mkdir -p /srv/sftp/sftpuser1/uploads
sudo chown sftpuser1:sftpusers /srv/sftp/sftpuser1/uploads
sudo chmod 755 /srv/sftp/sftpuser1/uploads

# Reload SSH
sudo systemctl reload sshd
```

## Restricting SFTP by IPv4 Address

```bash
# /etc/ssh/sshd_config

# Allow SFTP only from specific IPs
Match Group sftpusers Address 10.0.0.0/8,203.0.113.20
    ChrootDirectory /srv/sftp/%u
    ForceCommand    internal-sftp

# Block SFTP from other IPs (using AllowUsers with address restriction)
Match Group sftpusers Address !10.0.0.0/8
    DenyUsers *
```

```bash
# Alternatively, use TCP Wrappers (/etc/hosts.allow)
sshd: 10.0.0.0/8 203.0.113.20

# Or iptables (restrict port 22 to specific sources)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -s 203.0.113.20 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

## Using SFTP Clients

```bash
# Command-line sftp
sftp sftpuser1@203.0.113.10
sftp> ls
sftp> get remotefile.txt
sftp> put localfile.txt uploads/
sftp> bye

# Batch mode (non-interactive)
sftp -b - sftpuser1@203.0.113.10 << 'EOF'
put localfile.txt uploads/
ls uploads/
bye
EOF

# rsync over SFTP
rsync -avz -e ssh localdir/ sftpuser1@203.0.113.10:/uploads/

# scp (uses SSH transport)
scp localfile.txt sftpuser1@203.0.113.10:/uploads/

# With SSH key (preferred over password):
ssh-keygen -t ed25519 -f ~/.ssh/sftp_key
ssh-copy-id -i ~/.ssh/sftp_key sftpuser1@203.0.113.10
sftp -i ~/.ssh/sftp_key sftpuser1@203.0.113.10
```

## Conclusion

SFTP is the modern replacement for FTP — it uses SSH on a single port, requires no passive mode configuration, and encrypts all traffic automatically. Set up SFTP-only users with `Match Group` and `ForceCommand internal-sftp` in sshd_config, create chroot directories owned by root, and restrict access by source IP using `Address` in Match blocks. Public key authentication eliminates password exposure entirely.
