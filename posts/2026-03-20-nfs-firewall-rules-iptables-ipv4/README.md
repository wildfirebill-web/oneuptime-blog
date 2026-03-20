# How to Configure iptables Firewall Rules for NFS on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NFS, iptables, IPv4, Firewall, Security, RPC

Description: Write iptables rules to allow NFS traffic on IPv4, including portmapper, mountd, and NFS ports, with optional static port assignment for reliable firewall control.

## Introduction

NFS uses multiple ports: the well-known port 2049 for the NFS service, port 111 for rpcbind/portmapper, and dynamic ports for auxiliary services (mountd, statd, lockd) in NFSv3. Effective firewall rules require either static port assignment or using NFSv4 which consolidates everything on port 2049.

## NFSv4 Firewall Rules (Simplest)

```bash
# NFSv4 uses only two ports:
# 2049 — NFS service
# 111  — rpcbind (optional for NFSv4, but some implementations need it)

# Allow NFSv4 from trusted network
sudo iptables -A INPUT -p tcp --dport 2049 -s 10.0.0.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 111  -s 10.0.0.0/24 -j ACCEPT

# Block NFS from everywhere else
sudo iptables -A INPUT -p tcp --dport 2049 -j DROP
sudo iptables -A INPUT -p tcp --dport 111  -j DROP
sudo iptables -A INPUT -p udp --dport 111  -j DROP

# Allow established connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

## NFSv3 Static Port Assignment

```bash
# /etc/default/nfs-kernel-server (Debian/Ubuntu)
RPCMOUNTDOPTS="--port 2050"

# /etc/default/nfs-common
STATDOPTS="--port 2051 --outgoing-port 2052"

# /etc/default/nfs-kernel-server — lockd ports
# Use sysctl instead:
sudo sysctl -w fs.nfs.nlm_tcpport=2053
sudo sysctl -w fs.nfs.nlm_udpport=2053

# Persist sysctl settings
echo "fs.nfs.nlm_tcpport=2053" | sudo tee -a /etc/sysctl.conf
echo "fs.nfs.nlm_udpport=2053" | sudo tee -a /etc/sysctl.conf

sudo systemctl restart nfs-kernel-server rpcbind
```

## Complete NFSv3 Firewall Ruleset

```bash
#!/bin/bash
# NFS server iptables rules — with static ports

TRUSTED="10.0.0.0/24"
NFS_PORTS="111,2049,2050,2051,2052,2053"

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow NFS from trusted network only (TCP + UDP)
iptables -A INPUT -p tcp -m multiport --dports $NFS_PORTS \
  -s $TRUSTED -j ACCEPT
iptables -A INPUT -p udp -m multiport --dports $NFS_PORTS \
  -s $TRUSTED -j ACCEPT

# Drop NFS from everywhere else
iptables -A INPUT -p tcp -m multiport --dports $NFS_PORTS -j DROP
iptables -A INPUT -p udp -m multiport --dports $NFS_PORTS -j DROP

echo "NFS firewall rules applied"
```

## Restricting to Multiple Subnets

```bash
# Allow multiple trusted subnets

SUBNETS="10.0.0.0/24 192.168.1.0/24 203.0.113.20/32"

for subnet in $SUBNETS; do
  iptables -A INPUT -p tcp --dport 2049 -s $subnet -j ACCEPT
  iptables -A INPUT -p tcp --dport 111  -s $subnet -j ACCEPT
  iptables -A INPUT -p udp --dport 111  -s $subnet -j ACCEPT
done

iptables -A INPUT -p tcp --dport 2049 -j DROP
iptables -A INPUT -p tcp --dport 111  -j DROP
```

## Saving and Restoring Rules

```bash
# Save current rules
sudo iptables-save | sudo tee /etc/iptables/rules.v4

# Install iptables-persistent for automatic restore at boot
sudo apt install iptables-persistent

# Restore rules manually
sudo iptables-restore < /etc/iptables/rules.v4

# Verify rules are applied
sudo iptables -L INPUT -n -v | grep -E "111|2049|2050|2051"
```

## Conclusion

For NFSv4, only ports 2049 and 111 need firewall rules — keeping it simple. For NFSv3, assign static ports to mountd, statd, and lockd, then create explicit rules for each port. Always restrict NFS ports to known trusted subnets; NFS should never be accessible from the internet. Save rules with `iptables-persistent` for boot-time restoration.
