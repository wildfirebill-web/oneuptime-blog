# How to Set Up Samba as an Active Directory Domain Controller on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Samba, Active Directory, IPv4, Domain Controller, Kerberos, DNS, Linux

Description: Learn how to provision a Samba Active Directory Domain Controller on an IPv4-only network for Windows and Linux domain authentication.

---

Samba 4 can act as a full Active Directory Domain Controller (AD DC), providing Kerberos authentication, LDAP, DNS, and SMB services - all over IPv4. This enables Windows workstations and Linux clients to join a domain without Windows Server.

## Prerequisites

```bash
# Install Samba AD tools

apt install samba krb5-user winbind smbclient -y  # Debian/Ubuntu
# or
dnf install samba samba-dc krb5-workstation -y    # RHEL/Rocky

# Stop any existing Samba services
systemctl stop smbd nmbd winbind
```

## Provisioning the Domain

```bash
# Run the Samba domain provisioning tool
# This sets up the entire AD DS structure
samba-tool domain provision \
  --server-role=dc \
  --use-rfc2307 \
  --dns-backend=SAMBA_INTERNAL \
  --realm=EXAMPLE.COM \
  --domain=EXAMPLE \
  --adminpass='Admin@Password1!'

# realm: Kerberos realm (uppercase)
# domain: NetBIOS domain name
# adminpass: Administrator password (must meet complexity requirements)
```

Samba will create `/etc/samba/smb.conf` and `/var/lib/samba/private/`.

## Generated smb.conf (Excerpt)

```ini
# /etc/samba/smb.conf (auto-generated, do not edit directly for AD DC)

[global]
    workgroup = EXAMPLE
    realm = EXAMPLE.COM
    netbios name = DC1
    server role = active directory domain controller

    # Bind to the IPv4 address of this DC
    interfaces = lo 192.168.1.10/24
    bind interfaces only = yes

    dns forwarder = 8.8.8.8
```

## Configuring Kerberos

```bash
# Copy the generated krb5 config
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

## Starting the AD DC

```bash
# On systemd systems, use the samba service (not smbd/nmbd)
systemctl unmask samba-ad-dc
systemctl enable --now samba-ad-dc

# Verify it's running
samba-tool domain level show
```

## Testing the Domain Controller

```bash
# Get a Kerberos ticket for the administrator
kinit Administrator@EXAMPLE.COM

# List domain users
samba-tool user list

# Check DNS is working (SRV records for AD)
host -t SRV _ldap._tcp.example.com

# Check LDAP
ldapsearch -H ldap://192.168.1.10 -b "DC=example,DC=com" -W -D "administrator@example.com" "(objectClass=user)"
```

## Joining a Windows Client

On the Windows workstation:
1. Set DNS to `192.168.1.10` (the Samba DC).
2. Join the domain: **Control Panel → System → Change Settings → Domain → EXAMPLE.COM**.

## Key Takeaways

- `samba-tool domain provision` creates the entire AD DS structure including Kerberos, DNS, and LDAP services.
- Set `interfaces` to your IPv4 address to prevent the DC from binding to IPv6.
- Use `SAMBA_INTERNAL` DNS backend for simplicity; configure `dns forwarder` for internet resolution.
- Verify with `samba-tool domain level show` and `kinit Administrator` after provisioning.
