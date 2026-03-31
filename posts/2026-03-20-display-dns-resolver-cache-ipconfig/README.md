# How to Display the DNS Resolver Cache with ipconfig /displaydns

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Window, Networking, Ipconfig, DNS, Cache, Diagnostics

Description: Display the contents of the Windows DNS resolver cache using ipconfig /displaydns to inspect cached hostname-to-IP mappings and identify stale or incorrect DNS entries.

## Introduction

The Windows DNS resolver cache stores the results of recent DNS lookups. Displaying the cache lets you see which records are cached, what TTL remains, and identify incorrect entries that might be causing connectivity issues.

## Displaying the DNS Cache

```cmd
ipconfig /displaydns
```

Sample output for one entry:

```yaml
    google.com
    ----------------------------------------
    Record Name . . . . . : google.com
    Record Type . . . . . : 1
    Time To Live  . . . . : 263
    Data Length . . . . . : 4
    Section . . . . . . . : Answer
    A (Host) Record . . . : 142.250.80.46
```

## Understanding the Output Fields

| Field | Meaning |
|---|---|
| Record Type 1 | A record (IPv4 address) |
| Record Type 28 | AAAA record (IPv6 address) |
| Record Type 5 | CNAME record |
| Record Type 12 | PTR record (reverse DNS) |
| Time To Live | Seconds until this entry expires |
| A (Host) Record | Resolved IPv4 address |

## Filtering the Cache Output

The cache can be very long. Filter for a specific hostname:

```cmd
:: Find a specific entry
ipconfig /displaydns | findstr /i "google.com"

:: Show only A record lines
ipconfig /displaydns | findstr "A (Host)"

:: Count total cached entries
ipconfig /displaydns | findstr /c:"Record Name"
```

## Using PowerShell for Structured Cache Viewing

```powershell
# Get all DNS cache entries as objects

Get-DnsClientCache

# Filter for a specific name
Get-DnsClientCache | Where-Object {$_.Entry -like "*google*"}

# Show only IPv4 (A) records
Get-DnsClientCache | Where-Object {$_.Type -eq "A"}

# Sort by TTL to find entries about to expire
Get-DnsClientCache | Sort-Object TimeToLive | Select-Object Entry, Data, TimeToLive | Format-Table
```

## Finding Negative Cache Entries

Negative entries (NXDOMAIN responses) show `Record Type . . . : 0` or have no A record data. These indicate DNS queries that returned "not found":

```powershell
# Find negative cache entries
Get-DnsClientCache | Where-Object {$_.DataLength -eq 0}
```

## Exporting the Cache to CSV

```powershell
# Export all cache entries for analysis
Get-DnsClientCache | Select-Object Entry, Type, TimeToLive, Data |
    Export-Csv -Path C:\dns-cache.csv -NoTypeInformation
```

## Identifying Stale Entries

If a TTL is very high (e.g., 3600+) and you recently changed the DNS record:

```cmd
:: Find entries with TTL > 1 hour
ipconfig /displaydns | findstr "Time To Live"
```

If you see old IPs in the Data field, flush and re-resolve:

```cmd
ipconfig /flushdns
nslookup the-hostname.com
```

## Conclusion

`ipconfig /displaydns` reveals what Windows currently has cached, helping you understand why a hostname may resolve to an unexpected IP. Combine with `ipconfig /flushdns` to clear stale entries, and use PowerShell's `Get-DnsClientCache` when you need structured, filterable output.
