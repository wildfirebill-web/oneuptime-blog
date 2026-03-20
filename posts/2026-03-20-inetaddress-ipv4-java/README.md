# How to Use InetAddress to Work with IPv4 Addresses in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, InetAddress, IPv4, Inet4Address, Networking, DNS

Description: Learn how to use Java's InetAddress and Inet4Address classes to parse, validate, resolve, and work with IPv4 addresses programmatically.

## Parsing and Validating IPv4 Addresses

```java
import java.net.*;

public class InetAddressExample {
    public static boolean isValidIPv4(String s) {
        try {
            InetAddress addr = InetAddress.getByName(s);
            // Inet4Address is the subclass for IPv4
            return addr instanceof Inet4Address;
        } catch (UnknownHostException e) {
            return false;
        }
    }

    public static void main(String[] args) {
        String[] tests = {
            "192.168.1.1", "10.0.0.256", "::1", "8.8.8.8", "not-an-ip"
        };
        for (String t : tests) {
            System.out.printf("%-20s -> valid IPv4: %b%n", t, isValidIPv4(t));
        }
    }
}
```

## Creating InetAddress from Octets

```java
import java.net.*;

// Create from raw bytes (4 bytes for IPv4)
byte[] addr = { (byte)192, (byte)168, 1, 100 };
InetAddress ip = InetAddress.getByAddress(addr);
System.out.println(ip.getHostAddress());  // "192.168.1.100"

// Create Inet4Address specifically
Inet4Address ip4 = (Inet4Address) InetAddress.getByAddress(addr);
```

## DNS Resolution: Hostname to IPv4

```java
import java.net.*;

public class DNSResolver {
    // Get all IPv4 addresses for a hostname
    public static Inet4Address[] resolveIPv4(String hostname) throws UnknownHostException {
        InetAddress[] all = InetAddress.getAllByName(hostname);
        return java.util.Arrays.stream(all)
            .filter(a -> a instanceof Inet4Address)
            .toArray(Inet4Address[]::new);
    }

    public static void main(String[] args) throws Exception {
        Inet4Address[] ips = resolveIPv4("google.com");
        System.out.println("IPv4 addresses for google.com:");
        for (Inet4Address ip : ips) {
            System.out.println("  " + ip.getHostAddress());
        }
    }
}
```

## Reverse DNS Lookup

```java
import java.net.*;

// InetAddress.getByName() with an IP string + getHostName() for reverse lookup
InetAddress addr = InetAddress.getByName("8.8.8.8");
String hostname = addr.getHostName();  // dns.google
System.out.println("Reverse lookup: 8.8.8.8 -> " + hostname);
```

## IP Classification

```java
import java.net.*;

public class IpClassifier {
    public static void classify(String ipStr) throws UnknownHostException {
        InetAddress ip = InetAddress.getByName(ipStr);
        System.out.printf("%s:%n", ipStr);
        System.out.printf("  isLoopbackAddress:     %b%n", ip.isLoopbackAddress());
        System.out.printf("  isSiteLocalAddress:    %b%n", ip.isSiteLocalAddress());    // Private
        System.out.printf("  isLinkLocalAddress:    %b%n", ip.isLinkLocalAddress());    // 169.254.x.x
        System.out.printf("  isMulticastAddress:    %b%n", ip.isMulticastAddress());    // 224.x.x.x
        System.out.printf("  isAnyLocalAddress:     %b%n", ip.isAnyLocalAddress());     // 0.0.0.0
        System.out.printf("  isReachable(timeout):  %b%n", ip.isReachable(2000));
    }

    public static void main(String[] args) throws Exception {
        for (String ip : new String[]{"127.0.0.1", "192.168.1.1", "8.8.8.8", "224.0.0.1"}) {
            classify(ip);
        }
    }
}
```

## Converting IPv4 to Integer

```java
import java.net.*;
import java.nio.ByteBuffer;

public static int ipv4ToInt(String ipStr) throws UnknownHostException {
    Inet4Address addr = (Inet4Address) InetAddress.getByName(ipStr);
    return ByteBuffer.wrap(addr.getAddress()).getInt();
}

public static String intToIPv4(int n) throws UnknownHostException {
    byte[] bytes = ByteBuffer.allocate(4).putInt(n).array();
    return InetAddress.getByAddress(bytes).getHostAddress();
}

// Example
int num = ipv4ToInt("192.168.1.1");
System.out.printf("192.168.1.1 = %d%n", num);
System.out.printf("%d = %s%n", num, intToIPv4(num));
```

## Conclusion

Java's `InetAddress` and its subclass `Inet4Address` provide comprehensive IPv4 address handling: parsing, validation (via `instanceof Inet4Address`), DNS resolution (`getAllByName`), reverse lookup (`getHostName`), and classification (`isSiteLocalAddress`, `isLoopbackAddress`, etc.). Use `InetAddress.getAllByName()` with filtering for explicit IPv4-only resolution.
