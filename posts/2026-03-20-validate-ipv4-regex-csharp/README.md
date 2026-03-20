# How to Validate IPv4 Addresses Using Regex in C#

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: CSharp, Regex, IPv4, Validation, Networking, DotNet

Description: Learn how to validate IPv4 address strings using regular expressions in C#, with a compiled Regex pattern that enforces the 0-255 octet range and rejects leading zeros.

## Simple Pattern (Not Recommended)

```csharp
using System.Text.RegularExpressions;

// Naive: matches format but not octet range
var naive = new Regex(@"^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$");

Console.WriteLine(naive.IsMatch("192.168.1.1"));    // True
Console.WriteLine(naive.IsMatch("999.999.999.999")); // True - WRONG
```

## Strict Regex (Validates 0-255)

```csharp
using System;
using System.Text.RegularExpressions;

public static class Ipv4Validator
{
    // Compiled + cached for performance - Regex is thread-safe when compiled
    private static readonly Regex StrictIPv4 = new Regex(
        @"^(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)" +
        @"(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}$",
        RegexOptions.Compiled
    );

    public static bool IsValidIPv4(string? s)
    {
        if (s is null) return false;
        return StrictIPv4.IsMatch(s);
    }

    public static void Main()
    {
        (string ip, bool expected)[] tests =
        {
            ("192.168.1.1",     true),
            ("0.0.0.0",         true),
            ("255.255.255.255", true),
            ("256.0.0.1",       false),  // Octet > 255
            ("192.168.1",       false),  // Missing octet
            ("192.168.1.1.1",   false),  // Extra octet
            ("192.168.01.1",    false),  // Leading zero
            ("::1",             false),  // IPv6
            ("",                false),  // Empty
            (" 192.168.1.1",    false),  // Leading space
        };

        foreach (var (ip, expected) in tests)
        {
            bool result = IsValidIPv4(ip);
            string status = result == expected ? "PASS" : "FAIL";
            Console.WriteLine($"[{status}] {ip,-25} -> {result}");
        }
    }
}
```

## Using IPAddress.TryParse as the Preferred Alternative

```csharp
using System.Net;
using System.Net.Sockets;

// IPAddress.TryParse is the recommended approach for pure validation
bool IsValidIPv4Simple(string s)
{
    return IPAddress.TryParse(s, out var addr)
        && addr.AddressFamily == AddressFamily.InterNetwork;
}

Console.WriteLine(IsValidIPv4Simple("192.168.1.1"));   // True
Console.WriteLine(IsValidIPv4Simple("256.0.0.1"));     // False
Console.WriteLine(IsValidIPv4Simple("::1"));           // False (IPv6)
// Note: IPAddress.TryParse accepts some leading-zero forms - use regex when strict
```

## Extracting IPs from Text

```csharp
using System;
using System.Collections.Generic;
using System.Text.RegularExpressions;

public static class IpExtractor
{
    private static readonly Regex Finder = new Regex(
        @"\b(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)" +
        @"(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}\b",
        RegexOptions.Compiled
    );

    public static IEnumerable<string> ExtractIPs(string text)
    {
        foreach (Match m in Finder.Matches(text))
            yield return m.Value;
    }

    public static void Main()
    {
        string log = "[2026-03-20] Request from 192.168.1.50 to 10.0.0.1";
        foreach (string ip in ExtractIPs(log))
            Console.WriteLine(ip);
        // 192.168.1.50
        // 10.0.0.1
    }
}
```

## Data Annotation for ASP.NET Validation

```csharp
using System.ComponentModel.DataAnnotations;
using System.Text.RegularExpressions;

[AttributeUsage(AttributeTargets.Property | AttributeTargets.Parameter)]
public class ValidIPv4Attribute : ValidationAttribute
{
    private static readonly Regex Pattern = new Regex(
        @"^(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)" +
        @"(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}$",
        RegexOptions.Compiled
    );

    protected override ValidationResult? IsValid(object? value, ValidationContext ctx)
    {
        if (value is string s && Pattern.IsMatch(s))
            return ValidationResult.Success;
        return new ValidationResult("Invalid IPv4 address");
    }
}

// Usage on a model property:
// [ValidIPv4]
// public string ClientIp { get; set; }
```

## Conclusion

In C#, use `RegexOptions.Compiled` when validating many addresses in a hot path. The strict alternation `(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)` enforces the 0-255 range and rejects leading zeros. For simple validation without text extraction, `IPAddress.TryParse` combined with an `AddressFamily.InterNetwork` check is the idiomatic .NET approach. Use the regex when you need strict leading-zero rejection or IP extraction from freeform text.
