# How to Configure SMB Signing for IPv4 Network Security

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Samba, SMB Signing, IPv4, Security, MITM Protection, Window, Configuration

Description: Learn how to configure SMB signing on Samba and Windows to prevent man-in-the-middle attacks on IPv4 SMB connections.

---

SMB signing adds a cryptographic signature to each SMB packet, allowing the receiver to verify the packet came from the authenticated sender and wasn't modified in transit. Without signing, attackers on the same IPv4 network can perform NTLM relay and man-in-the-middle attacks.

## SMB Signing Options

| Setting | Behavior |
|---------|----------|
| `disabled` | Never sign (vulnerable to MITM) |
| `auto` | Sign if the other end also supports signing |
| `mandatory` | Always sign; reject unsigned connections |

## Configuring Samba Server Signing

```ini
# /etc/samba/smb.conf

[global]
    workgroup = MYORG
    server signing = mandatory     # Require signing on all SMB connections

    # Also set for the client portion of Samba (used when Samba connects to other servers)
    client signing = mandatory
```

Acceptable values:
- `disabled` or `no` - Don't sign
- `auto` or `if_required` - Sign if peer requests it
- `mandatory` or `required` - Always sign, reject if peer doesn't

```bash
# Test the configuration

testparm

# Reload Samba
systemctl reload smb
```

## Verifying Signing is Active

```bash
# Connect with smbclient and check signing status
smbclient //192.168.1.10/data -U user%password

# The connection message will include signing status:
# "Session Setup AndX Response: ok, Signing: Active"
```

## Windows Server and Client Configuration

Group Policy settings for SMB signing:

```text
Computer Configuration → Windows Settings → Security Settings
→ Local Policies → Security Options:

# Server:
"Microsoft network server: Digitally sign communications (always)" = Enabled
"Microsoft network server: Digitally sign communications (if client agrees)" = Enabled

# Client (workstation):
"Microsoft network client: Digitally sign communications (always)" = Enabled
"Microsoft network client: Digitally sign communications (if server agrees)" = Enabled
```

## Registry Settings (Windows)

```powershell
# Enable required SMB signing on Windows Server
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" `
  -Name "RequireSecuritySignature" -Value 1

# Enable required SMB signing on Windows client
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters" `
  -Name "RequireSecuritySignature" -Value 1

# Check current signing status
Get-SmbServerConfiguration | Select-Object -Property RequireSecuritySignature
Get-SmbClientConfiguration | Select-Object -Property RequireSecuritySignature
```

## Performance Considerations

SMB signing adds cryptographic overhead (approximately 5-15% CPU increase for high-throughput shares). For high-performance file servers, consider:
- Enabling SMB 3.0+ which uses AES-CMAC signing (faster than SMB 2 HMAC-SHA256).
- Using dedicated network hardware with offload capabilities.

## Key Takeaways

- Set `server signing = mandatory` in `smb.conf` to require signed connections from all SMB clients.
- Signing prevents NTLM relay attacks where an attacker intercepts and replays authentication credentials on the IPv4 network.
- SMB 3.x provides encryption in addition to signing - enable it for highly sensitive shares.
- Enabling `mandatory` signing will break connections from very old clients that don't support signing.
