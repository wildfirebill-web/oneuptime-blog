# How to Export NFS Shares to IPv4 Subnets Using CIDR Notation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NFS, IPv4, CIDR, Subnet, Exports, Access Control

Description: Use CIDR notation in /etc/exports to grant NFS access to entire IPv4 subnets, configure different permissions per subnet, and manage multi-site NFS access control.

## Introduction

NFS exports support CIDR notation for granting access to entire subnets at once, eliminating the need to list individual IPs. This is essential for environments with many clients or dynamically assigned IPs within known ranges.

## CIDR Notation in /etc/exports

```bash
# /etc/exports

# Allow entire /24 subnet (256 addresses)
/srv/data  192.168.1.0/24(rw,sync,no_subtree_check)

# Allow /16 subnet (65536 addresses)
/srv/shared  10.0.0.0/16(rw,sync,no_subtree_check)

# Separate permissions per subnet
/srv/production  10.0.1.0/24(rw,sync,no_subtree_check) \
                 10.0.2.0/24(ro,sync,no_subtree_check)

# Multiple subnets with full options
/var/www/html  \
  10.0.0.0/8(ro,sync,no_subtree_check,root_squash) \
  172.16.0.0/12(ro,sync,no_subtree_check,root_squash)
```

## Subnet Notation Formats

NFS exports support multiple formats for specifying clients:

```bash
# CIDR notation (preferred)
/srv/share  192.168.1.0/24(rw,sync)

# Wildcard hostname (not IP-based, but common)
/srv/share  *.internal.example.com(rw,sync)

# Network/netmask notation (older format, still works)
/srv/share  192.168.1.0/255.255.255.0(rw,sync)

# Combination: specific IP gets rw, subnet gets ro
/srv/configs  10.0.0.5(rw,sync,no_subtree_check) 10.0.0.0/24(ro,sync)
```

## Multi-Site Configuration

```bash
# /etc/exports — multi-office setup

# Headquarters (read-write)
/srv/company-files  10.1.0.0/24(rw,sync,no_subtree_check,root_squash)

# Branch offices (read-only)
/srv/company-files  10.2.0.0/24(ro,sync,no_subtree_check) \
                    10.3.0.0/24(ro,sync,no_subtree_check)

# DMZ servers (specific files only, read-only)
/srv/public-assets  203.0.113.0/28(ro,sync,no_subtree_check,all_squash)
```

## Applying and Verifying

```bash
# Apply updated exports
sudo exportfs -ra

# List active exports with client restrictions
sudo exportfs -v
# Expected:
# /srv/data      192.168.1.0/24(rw,sync,wdelay,hide,no_subtree_check,...)

# Verify a specific IP would be allowed (from the client)
showmount -e 203.0.113.10

# Test mount from an allowed subnet IP:
sudo mount -t nfs 203.0.113.10:/srv/data /mnt/test
ls /mnt/test
sudo umount /mnt/test
```

## Security Considerations

```bash
# Avoid overly broad subnets — be as specific as possible
# Instead of:
/srv/sensitive  0.0.0.0/0(rw,sync)   # NEVER do this

# Use:
/srv/sensitive  10.0.1.0/24(rw,sync,no_subtree_check,root_squash)

# Verify no subnet is exporting to the entire internet:
sudo exportfs -v | grep "0.0.0.0\|/0"  # Should return nothing

# Block NFS ports from internet using iptables
sudo iptables -A INPUT -p tcp -m multiport --dports 111,2049 \
  -s 10.0.0.0/8 -j ACCEPT
sudo iptables -A INPUT -p tcp -m multiport --dports 111,2049 -j DROP
```

## Conclusion

CIDR notation in `/etc/exports` allows granting NFS access to entire subnets with a single entry. Use `subnet/prefix` format (e.g., `10.0.0.0/24`), apply different permission sets per subnet, and run `exportfs -ra` to activate changes. Always use the most restrictive subnet that covers your clients, and block NFS ports at the firewall to prevent internet exposure.
