# How to Configure iptables Firewall Rules for FTP on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: iptables, FTP, IPv4, Firewall, Active Mode, Passive Mode

Description: Write iptables rules to allow FTP command and data connections for both active and passive modes on IPv4, including the nf_conntrack_ftp module for automatic port tracking.

## Introduction

FTP requires special firewall handling because the data channel uses dynamically negotiated ports. iptables rules for FTP must account for both active mode (server initiates port 20 connection) and passive mode (client connects to ephemeral port). The `nf_conntrack_ftp` kernel module automates most of this.

## FTP Port Overview

```text
Active Mode:
  Control: Client → Server:21
  Data:    Server:20 → Client (random high port)

Passive Mode:
  Control: Client → Server:21
  Data:    Client → Server (random high port or pasv_min:pasv_max range)
```

## Using nf_conntrack_ftp (Recommended)

```bash
# Load the FTP connection tracking helper

sudo modprobe nf_conntrack_ftp

# Make it persistent across reboots
echo "nf_conntrack_ftp" | sudo tee -a /etc/modules

# Verify it's loaded
lsmod | grep nf_conntrack_ftp

# With nf_conntrack_ftp, iptables RELATED state handles FTP data ports:
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

## Complete iptables Ruleset for FTP Server

```bash
#!/bin/bash
# FTP server iptables rules

# Allow established connections
iptables -A INPUT  -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow FTP command port (inbound)
iptables -A INPUT -p tcp --dport 21 -j ACCEPT

# Active mode: allow data from port 20 outbound
# (nf_conntrack_ftp handles the reply via RELATED state)
iptables -A OUTPUT -p tcp --sport 20 -j ACCEPT

# Passive mode: allow incoming connections to passive port range
# (only needed if using static pasv_min/pasv_max and not conntrack)
iptables -A INPUT -p tcp --dport 30000:31000 -j ACCEPT

# Log and drop everything else
iptables -A INPUT -j LOG --log-prefix "FW-DROP: "
iptables -A INPUT -j DROP
```

## Restricting FTP Access to Specific IPs

```bash
# Allow FTP only from trusted IPs
iptables -A INPUT -p tcp --dport 21 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 21 -s 203.0.113.20 -j ACCEPT
iptables -A INPUT -p tcp --dport 21 -j DROP    # Block all others

# Allow passive ports only from same trusted IPs
iptables -A INPUT -p tcp --dport 30000:31000 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 30000:31000 -j DROP
```

## Saving iptables Rules

```bash
# Debian/Ubuntu
sudo apt install iptables-persistent
sudo netfilter-persistent save

# RHEL/CentOS
sudo service iptables save
# or
sudo iptables-save > /etc/sysconfig/iptables

# Verify saved rules
sudo cat /etc/iptables/rules.v4
```

## Testing the Rules

```bash
# From a test client:
ftp 203.0.113.10

# From internal allowed IP:
ftp -p 203.0.113.10   # Passive mode

# Check connection tracking state
sudo cat /proc/net/nf_conntrack | grep ftp

# View iptables hit counts
sudo iptables -L INPUT -v -n | grep -E "21|30000"
```

## Conclusion

Load `nf_conntrack_ftp` to let iptables automatically handle FTP data connections via the `RELATED` state. Add an INPUT rule for port 21 and your passive port range. Restrict source IPs with `-s` to limit FTP access to trusted clients. Save rules with `iptables-persistent` or the distribution equivalent for persistence across reboots.
