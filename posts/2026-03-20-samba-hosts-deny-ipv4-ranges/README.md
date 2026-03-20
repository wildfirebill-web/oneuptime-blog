# How to Block IPv4 Ranges with Samba hosts deny

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Samba, IPv4, Security, hosts deny, Access Control, SMB, Configuration

Description: Learn how to use Samba's hosts allow and hosts deny directives to block specific IPv4 addresses and CIDR ranges from accessing SMB shares.

---

Samba's `hosts allow` and `hosts deny` directives provide IP-based access control at the server and share level. These are particularly useful for blocking known malicious IP ranges or restricting shares to specific subnets.

## Global Access Control (smb.conf)

```ini
# /etc/samba/smb.conf

[global]
    workgroup = MYNET
    server string = Samba Server

    # Allow access only from internal IPv4 networks
    hosts allow = 192.168.1.0/24 10.0.0.0/8 127.0.0.1

    # Deny access from known bad IPv4 ranges
    hosts deny = 198.51.100.0/24 203.0.113.50 0.0.0.0/0

    # Evaluation order:
    # 1. hosts allow is checked first
    # 2. If not in hosts allow, hosts deny is checked
    # 3. If in neither: depends on which list is more specific
```

> Samba evaluates `hosts allow` and `hosts deny` with "most specific match wins". Listing `0.0.0.0/0` in `hosts deny` with a specific subnet in `hosts allow` effectively creates an allowlist.

## Per-Share Access Control

Override global settings for individual shares:

```ini
[global]
    # Default: deny all
    hosts deny = ALL

[public-share]
    path = /srv/samba/public
    browseable = yes
    guest ok = yes
    # Allow only internal LAN for this share
    hosts allow = 192.168.1.0/24

[sensitive-data]
    path = /srv/samba/sensitive
    valid users = @data-team
    # Further restrict to specific admin workstations
    hosts allow = 192.168.1.5 192.168.1.6 192.168.1.7
    hosts deny = ALL
```

## Blocking an Entire Subnet

```ini
[global]
    # Allow all internal networks
    hosts allow = 10.0.0.0/8 192.168.0.0/16

    # Explicitly block a problematic IPv4 range
    hosts deny = 192.168.1.200/29   # Block .200-.207

    # Block all other external traffic
    hosts deny = ALL
```

## Reloading Samba

```bash
# Test smb.conf syntax
testparm

# Reload Samba without restarting (applies most changes)
smbcontrol all reload-config

# Or restart for a full reload
systemctl restart smb nmb
```

## Verifying Access Control

```bash
# Test from an allowed IP (should succeed)
smbclient //192.168.1.10/public -U guest%

# Test from a denied IP (should fail with NT_STATUS_ACCESS_DENIED)
# Simulate by temporarily blocking with iptables
iptables -A OUTPUT -p tcp --sport 445 -d 198.51.100.5 -j DROP

# Check Samba log for denied connections
grep "denied" /var/log/samba/log.smbd | tail -20
```

## Key Takeaways

- `hosts allow` and `hosts deny` can be applied globally in `[global]` or per-share.
- For allowlisting, set `hosts deny = ALL` and list permitted IPs in `hosts allow`.
- Samba supports full CIDR notation (e.g., `192.168.1.0/24`) in both directives.
- Run `testparm` before reloading to catch configuration errors.
