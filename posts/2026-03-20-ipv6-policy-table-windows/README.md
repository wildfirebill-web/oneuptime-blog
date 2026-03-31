# How to Configure the IPv6 Policy Table on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Policy Table, Window, RFC 6724, Address Selection, PowerShell

Description: Configure the IPv6 address selection policy table on Windows using netsh and PowerShell to control source and destination address preferences for dual-stack networking.

## Windows IPv6 Policy Table

Windows implements RFC 6724 address selection with a configurable policy table managed through `netsh` and PowerShell. The table controls both source address selection and destination address sorting in `getaddrinfo()`.

## Viewing the Current Policy Table

```powershell
# PowerShell: view the address selection policy table

Get-NetIPv6Protocol | Select-Object -Property *

# View current prefix policies (Windows Server 2012+)
netsh interface ipv6 show prefixpolicies

# Output example:
# Active State   : enabled
# Prefix                          Precedence  Label
# --------------------------------  ----------  -----
# ::1/128                              50         0
# ::/0                                 40         1
# ::ffff:0:0/96                        35         4
# 2002::/16                            30         2
# 2001::/32                             5         5
# fc00::/7                              3        13
# ::/96                                 1         3
# fec0::/10                             1        11
# 3ffe::/16                             1        12
```

## Adding and Modifying Policy Entries

```powershell
# Add a prefix policy entry
# netsh interface ipv6 add prefixpolicy <prefix> <precedence> <label>

# Example: prefer IPv4 over IPv6 (raise IPv4-mapped precedence)
netsh interface ipv6 add prefixpolicy ::ffff:0:0/96 precedence=100 label=4

# Verify the change
netsh interface ipv6 show prefixpolicies

# Delete a custom entry (revert to default)
netsh interface ipv6 delete prefixpolicy ::ffff:0:0/96

# Reset all prefix policies to RFC 6724 defaults
netsh interface ipv6 reset
```

## Preferring IPv4 on Windows

```powershell
# Method 1: Raise IPv4-mapped precedence via netsh
netsh interface ipv6 add prefixpolicy ::ffff:0:0/96 precedence=100 label=4

# Method 2: Disable IPv6 on specific adapters
# (nuclear option - removes IPv6 entirely from adapter)
Disable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6

# Method 3: Prefer IPv4 via registry (legacy)
# HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters
# DisabledComponents = 0x20 (prefer IPv4 over IPv6)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name "DisabledComponents" `
    -Value 0x20 `
    -Type DWord

# Note: registry method requires reboot; netsh takes effect immediately
```

## Preferring IPv6 (Restoring Defaults)

```powershell
# Remove any IPv4 preference override
netsh interface ipv6 delete prefixpolicy ::ffff:0:0/96

# Or reset all policies to defaults
netsh interface ipv6 reset

# Verify IPv6 is now preferred
# Test: does getaddrinfo return IPv6 first for dual-stack host?
[System.Net.Dns]::GetHostAddresses("example.com") | ForEach-Object {
    Write-Host "$($_.AddressFamily): $($_.IPAddressToString)"
}
# Expected: InterNetworkV6 entries appear before InterNetwork (IPv4)
```

## ULA Address Handling

```powershell
# Check if ULA addresses (fc00::/7) have correct label
netsh interface ipv6 show prefixpolicies | Select-String "fc00"

# ULA label should be 13 - same as ULA sources
# This keeps ULA-to-ULA communication preferred

# Add explicit ULA entry if missing
netsh interface ipv6 add prefixpolicy fc00::/7 precedence=3 label=13
```

## Scripting Policy Management

```powershell
# PowerShell script to apply RFC 6724 defaults
# Useful after reset or on new deployments

function Set-IPv6DefaultPolicies {
    $policies = @(
        @{ Prefix = "::1/128";         Precedence = 50; Label = 0 },
        @{ Prefix = "::/0";            Precedence = 40; Label = 1 },
        @{ Prefix = "::ffff:0:0/96";   Precedence = 35; Label = 4 },
        @{ Prefix = "2002::/16";       Precedence = 30; Label = 2 },
        @{ Prefix = "2001::/32";       Precedence =  5; Label = 5 },
        @{ Prefix = "fc00::/7";        Precedence =  3; Label = 13 },
        @{ Prefix = "::/96";           Precedence =  1; Label = 3 },
        @{ Prefix = "fec0::/10";       Precedence =  1; Label = 11 },
        @{ Prefix = "3ffe::/16";       Precedence =  1; Label = 12 }
    )

    foreach ($p in $policies) {
        netsh interface ipv6 add prefixpolicy `
            "$($p.Prefix)" `
            "precedence=$($p.Precedence)" `
            "label=$($p.Label)" | Out-Null
        Write-Host "Set: $($p.Prefix) precedence=$($p.Precedence) label=$($p.Label)"
    }
}

Set-IPv6DefaultPolicies
```

## Testing Address Selection on Windows

```powershell
# Test which source address Windows would use
# PowerShell: connect a UDP socket (no actual traffic sent)
$socket = New-Object System.Net.Sockets.Socket(
    [System.Net.Sockets.AddressFamily]::InterNetworkV6,
    [System.Net.Sockets.SocketType]::Dgram,
    [System.Net.Sockets.ProtocolType]::Udp
)
$remote = [System.Net.IPEndPoint]::new([System.Net.IPAddress]::Parse("2001:db8::1"), 80)
$socket.Connect($remote)
Write-Host "Selected source: $($socket.LocalEndPoint)"
$socket.Close()

# Check destination sort order
[System.Net.Dns]::GetHostEntry("dual-stack-host.example.com").AddressList | ForEach-Object {
    Write-Host "$($_.AddressFamily): $($_.IPAddressToString)"
}
```

## Group Policy for Enterprise Deployment

```powershell
# Apply IPv6 policy settings via Group Policy (GPO)
# Path: Computer Configuration > Windows Settings > Scripts (Startup)

# Startup script content (deploy via GPO):
$script = @'
netsh interface ipv6 add prefixpolicy ::ffff:0:0/96 precedence=100 label=4
'@
# Save to SYSVOL and reference in GPO startup scripts

# Alternatively, use GPO Registry settings:
# HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters
# DisabledComponents values:
# 0x00 = All IPv6 enabled (default)
# 0x10 = Disable IPv6 on non-tunnel interfaces
# 0x20 = Prefer IPv4 over IPv6 in prefix policies
# 0xFF = Disable IPv6 completely
```

## Conclusion

Windows IPv6 policy table management uses `netsh interface ipv6 add prefixpolicy` to set precedence and label values. To prefer IPv4, raise `::ffff:0:0/96` precedence above 40 (the default IPv6 global precedence). To restore RFC 6724 defaults, use `netsh interface ipv6 reset` or re-add the standard entries. Enterprise environments can distribute policy via Group Policy startup scripts or the `DisabledComponents` registry value. Test changes using PowerShell's `System.Net.Sockets.Socket.Connect()` to observe source address selection without sending traffic.
