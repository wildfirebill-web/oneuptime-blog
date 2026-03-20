# How to Configure DHCP Exclusion Ranges

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Networking, Exclusion Ranges, IP Addressing, sysadmin

Description: DHCP exclusion ranges prevent the server from dynamically assigning specific IP addresses within a scope, reserving them for statically configured devices like servers, printers, and network equipment.

## What Are Exclusion Ranges?

Within a DHCP scope, the entire range between the start and end IP is considered available. Exclusion ranges carve out portions of that range that the server will never offer. This is the complement to reservations — exclusions protect addresses that may be statically assigned to any device.

## Best Practice: Separate Static and Dynamic Ranges

```
Subnet: 192.168.1.0/24
.1–.9:   Network equipment (router, switches) — outside pool
.10–.49: Servers and printers — statically assigned (excluded or below pool)
.50–.200: Dynamic DHCP pool
.201–.254: Reserved for future static use
```

## ISC dhcpd: Exclusions via Range Definition

In ISC dhcpd, exclusions are implemented by defining multiple range statements within a pool, leaving gaps:

```
subnet 192.168.1.0 netmask 255.255.255.0 {
    # Skip .1–.49 (network equipment and servers — not in range)
    range 192.168.1.50 192.168.1.200;
    # Skip .201–.254 (future static use — above range)
    option routers 192.168.1.1;
}
```

For exclusions in the middle of a range, use multiple range statements:

```
subnet 10.0.10.0 netmask 255.255.255.0 {
    # Exclude .50–.59 in the middle (printer row)
    range 10.0.10.10 10.0.10.49;
    range 10.0.10.60 10.0.10.200;
    option routers 10.0.10.1;
}
```

## Windows Server PowerShell: Exclusion Ranges

```powershell
# Add an exclusion range to an existing scope
Add-DhcpServerv4ExclusionRange `
    -ScopeId 192.168.1.0 `
    -StartRange 192.168.1.1 `
    -EndRange 192.168.1.49

# Add another exclusion for .201–.254
Add-DhcpServerv4ExclusionRange `
    -ScopeId 192.168.1.0 `
    -StartRange 192.168.1.201 `
    -EndRange 192.168.1.254

# View all exclusion ranges
Get-DhcpServerv4ExclusionRange -ScopeId 192.168.1.0

# Remove an exclusion range
Remove-DhcpServerv4ExclusionRange `
    -ScopeId 192.168.1.0 `
    -StartRange 192.168.1.201 `
    -EndRange 192.168.1.254
```

## dnsmasq: Implicit Exclusions

In dnsmasq, the range statement implicitly excludes anything outside the start-end bounds:

```
# Only .50 to .200 are in the dynamic pool; all others are implicitly excluded
dhcp-range=192.168.1.50,192.168.1.200,255.255.255.0,24h
```

For mid-range exclusions, add static hosts outside the pool or use tag-based filtering.

## Python: Documenting Reserved vs Available Addresses

```python
import ipaddress

subnet = ipaddress.IPv4Network("192.168.1.0/24")
dynamic_pool = range(50, 201)  # .50 to .200
excluded = range(1, 50)        # .1 to .49

print("Address allocation plan:")
for i in range(1, 255):
    addr = ipaddress.IPv4Address(f"192.168.1.{i}")
    if i in excluded:
        status = "STATIC/RESERVED"
    elif i in dynamic_pool:
        status = "DYNAMIC POOL"
    else:
        status = "UNALLOCATED"
    if i % 50 == 0 or i <= 3:  # Print select addresses
        print(f"  {addr}  {status}")
```

## Key Takeaways

- Exclusion ranges protect addresses from dynamic assignment.
- In ISC dhcpd, exclusions are implicit — only addresses within `range` statements are offered.
- Windows Server has explicit `Add-DhcpServerv4ExclusionRange` cmdlets.
- Best practice: keep static assignments (.1–.49), dynamic pool (.50–.200), and future reserve (.201–.254) clearly separated.
