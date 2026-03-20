# How IPv6 Privacy Extensions Work on iOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Privacy Extensions, IOS, IPhone, IPad, Apple, Security

Description: A guide to understanding IPv6 privacy extensions on iOS and iPadOS, including Apple's Private Wi-Fi Address feature, how it relates to IPv6 privacy, and how developers should handle IPv6 addresses...

iOS implements IPv6 privacy through two complementary features: IPv6 privacy extensions (RFC 8981) for temporary IPv6 addresses, and Private Wi-Fi Address (randomized MAC address) which prevents EUI-64 address derivation entirely. Together, these make iOS one of the most privacy-preserving mobile platforms for IPv6.

## Apple's IPv6 Privacy Approach

```text
iOS 14+ IPv6 Privacy Stack:
├── Private Wi-Fi Address (MAC randomization)
│   - Random MAC per Wi-Fi network
│   - Prevents EUI-64 interface ID derivation from hardware MAC
│   - Enabled by default since iOS 14
│
└── IPv6 Privacy Extensions (RFC 8981)
    - Random interface ID generation
    - Temporary addresses for outbound connections
    - Stable privacy address (RFC 7217) for inbound
    - Addresses rotate periodically
```

## Checking IPv6 Address on iOS

Without direct terminal access, checking your IPv6 address on iOS requires:

**Via Settings:**
1. Settings > Wi-Fi > tap your network name > tap the `ⓘ` icon
2. Scroll down to "IPv6 Address"
3. You should see multiple IPv6 addresses (temporary + stable)

**Via third-party apps:**
```text
Apps like "Network Analyzer" or "iStat" show all IPv6 addresses
including temporary (privacy extension) addresses
```

**Via a website:**
```text
Visit https://test-ipv6.com or https://ipv6.icanhazip.com in Safari
The displayed address will be your current temporary IPv6 address
```

## Private Wi-Fi Address and IPv6

```bash
# Private Wi-Fi Address (MAC randomization) is key for IPv6 privacy

# When MAC is randomized, EUI-64 derivation produces a random-looking address
# rather than a hardware-traceable one

# What iOS does per Wi-Fi network:
# 1. Generate a random MAC for this specific SSID
# 2. Keep the same random MAC for this SSID (rotating periodically)
# 3. Generate IPv6 addresses from this randomized MAC via EUI-64
# OR
# 4. Use RFC 8981 temporary addresses (independent of MAC)

# In iOS 14+, both mechanisms work together
# The result: no consistent IPv6 address across networks or sessions
```

## Disabling/Enabling Private Address per Network

**Settings > Wi-Fi > [Network Name] > Private Wi-Fi Address**

```text
Rotating: Changes MAC periodically (iOS 18+)
Fixed: Uses consistent random MAC for this network
Off: Uses device's real MAC address (not recommended)
```

When Private Wi-Fi Address is OFF, the IPv6 address may contain the real MAC address as EUI-64. When ON, the MAC is randomized, preventing hardware tracking even at the IPv6 address level.

## iOS for Developers: Handling IPv6

Apple requires all iOS apps to support IPv6:

```swift
// Swift: Don't hardcode IPv4 addresses in iOS apps
// Apple's App Store review tests apps on IPv6-only networks

// BAD: Hardcoded IPv4 address
let serverURL = URL(string: "http://192.168.1.100/api")!

// GOOD: Use hostnames (DNS handles IPv4/IPv6 transparently)
let serverURL = URL(string: "https://api.example.com/")!

// GOOD: Use URLSession which handles Happy Eyeballs automatically
let session = URLSession.shared
let task = session.dataTask(with: serverURL) { data, response, error in
    // URLSession automatically prefers IPv6 if available (Happy Eyeballs)
    // No special IPv6 handling needed
}
task.resume()
```

```swift
// Swift: Getting the device's IPv6 addresses programmatically
import Network

func getIPv6Addresses() -> [String] {
    var addresses: [String] = []
    var ifaddr: UnsafeMutablePointer<ifaddrs>?

    guard getifaddrs(&ifaddr) == 0 else { return addresses }
    defer { freeifaddrs(ifaddr) }

    var ptr = ifaddr
    while ptr != nil {
        let addr = ptr!.pointee.ifa_addr
        if addr?.pointee.sa_family == UInt8(AF_INET6) {
            var hostname = [CChar](repeating: 0, count: Int(NI_MAXHOST))
            if getnameinfo(addr, socklen_t(addr!.pointee.sa_len),
                          &hostname, socklen_t(hostname.count),
                          nil, 0, NI_NUMERICHOST) == 0 {
                let address = String(cString: hostname)
                // Filter out link-local addresses
                if !address.hasPrefix("fe80") {
                    addresses.append(address)
                }
            }
        }
        ptr = ptr!.pointee.ifa_next
    }
    return addresses
}
```

## IPv6-Only Network Compatibility (Required for App Store)

```swift
// Apple requires apps to work on IPv6-only networks (since 2016)

// Check network connectivity (works for both IPv4 and IPv6)
import Network

let monitor = NWPathMonitor()
monitor.pathUpdateHandler = { path in
    if path.status == .satisfied {
        if path.supportsIPv6 {
            print("IPv6 is available")
        }
        if path.usesInterfaceType(.wifi) {
            print("Connected via Wi-Fi")
        }
    }
}
let queue = DispatchQueue(label: "NetworkMonitor")
monitor.start(queue: queue)
```

## Verifying IPv6-Only App Compatibility

```bash
# Apple provides a tool to test IPv6-only networks
# In Xcode: Window > Devices and Simulators
# Use iPhone/iPad "Internet Sharing" configured for IPv6-only

# Or in macOS: System Settings > Sharing > Internet Sharing
# Configure the shared network as IPv6-only for testing

# Test your app connects correctly on the IPv6-only network
# If it fails, check for hardcoded IPv4 addresses or
# direct socket calls that don't use getaddrinfo()
```

iOS provides comprehensive IPv6 privacy through randomized MAC addresses (Private Wi-Fi Address) and RFC 8981 privacy extensions, making it impossible to track iOS devices across networks using IPv6 addresses. App developers must ensure their apps work on IPv6-only networks (an App Store requirement), using hostname-based connections and URLSession rather than hardcoded IP addresses.
