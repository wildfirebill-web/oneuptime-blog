# How to Build a Simple DNS Lookup Tool in Java for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, DNS, IPv4, InetAddress, Networking, DNS Lookup

Description: Learn how to build a command-line DNS lookup tool in Java that resolves hostnames to IPv4 addresses and performs reverse lookups.

## Complete DNS Lookup Tool

```java
import java.net.*;
import java.util.*;

public class DnsLookupTool {
    public static void main(String[] args) {
        if (args.length == 0) {
            System.out.println("Usage: java DnsLookupTool <hostname|ip> [hostname2 ...]");
            System.exit(1);
        }

        for (String target : args) {
            System.out.printf("=== %s ===%n", target);
            if (looksLikeIP(target)) {
                reverseLookup(target);
            } else {
                forwardLookup(target);
            }
            System.out.println();
        }
    }

    private static boolean looksLikeIP(String s) {
        return s.matches("\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}");
    }

    private static void forwardLookup(String hostname) {
        try {
            InetAddress[] addresses = InetAddress.getAllByName(hostname);
            System.out.printf("Forward lookup for: %s%n", hostname);

            int ipv4Count = 0, ipv6Count = 0;
            for (InetAddress addr : addresses) {
                if (addr instanceof Inet4Address) {
                    System.out.printf("  A record:    %s%n", addr.getHostAddress());
                    ipv4Count++;
                } else {
                    System.out.printf("  AAAA record: %s%n", addr.getHostAddress());
                    ipv6Count++;
                }
            }
            System.out.printf("Total: %d IPv4, %d IPv6 addresses%n", ipv4Count, ipv6Count);

        } catch (UnknownHostException e) {
            System.err.printf("Not found: %s (%s)%n", hostname, e.getMessage());
        }
    }

    private static void reverseLookup(String ip) {
        try {
            InetAddress addr = InetAddress.getByName(ip);
            System.out.printf("Reverse lookup for: %s%n", ip);

            // getCanonicalHostName() triggers reverse DNS (PTR record)
            String hostname = addr.getCanonicalHostName();
            if (hostname.equals(ip)) {
                System.out.println("  No PTR record found");
            } else {
                System.out.printf("  PTR record: %s%n", hostname);
            }

            // Classify the address
            System.out.printf("  isLoopback:    %b%n", addr.isLoopbackAddress());
            System.out.printf("  isPrivate:     %b%n", addr.isSiteLocalAddress());
            System.out.printf("  isMulticast:   %b%n", addr.isMulticastAddress());
            System.out.printf("  isReachable:   ");
            System.out.printf("%b%n", addr.isReachable(2000));

        } catch (Exception e) {
            System.err.printf("Error looking up %s: %s%n", ip, e.getMessage());
        }
    }
}
```

## Caching DNS Resolver

```java
import java.net.*;
import java.util.*;
import java.util.concurrent.*;

public class CachingDnsResolver {
    private final Map<String, List<String>> cache = new ConcurrentHashMap<>();
    private final Map<String, Long> cacheTimestamps = new ConcurrentHashMap<>();
    private final long ttlMillis;

    public CachingDnsResolver(long ttlSeconds) {
        this.ttlMillis = ttlSeconds * 1000;
    }

    public List<String> resolveIPv4(String hostname) throws UnknownHostException {
        // Check cache
        Long timestamp = cacheTimestamps.get(hostname);
        if (timestamp != null && System.currentTimeMillis() - timestamp < ttlMillis) {
            System.out.println("[cache hit] " + hostname);
            return cache.get(hostname);
        }

        // Perform DNS lookup
        InetAddress[] addresses = InetAddress.getAllByName(hostname);
        List<String> ipv4s = new ArrayList<>();
        for (InetAddress addr : addresses) {
            if (addr instanceof Inet4Address) {
                ipv4s.add(addr.getHostAddress());
            }
        }

        if (!ipv4s.isEmpty()) {
            cache.put(hostname, ipv4s);
            cacheTimestamps.put(hostname, System.currentTimeMillis());
        }

        return ipv4s;
    }

    public static void main(String[] args) throws Exception {
        CachingDnsResolver resolver = new CachingDnsResolver(60);  // 60-second TTL

        // First call: DNS lookup
        System.out.println(resolver.resolveIPv4("google.com"));
        // Second call: cache hit
        System.out.println(resolver.resolveIPv4("google.com"));
    }
}
```

## Parallel DNS Resolution

```java
import java.net.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.*;

public class ParallelDnsResolver {
    public static Map<String, List<String>> resolveAll(List<String> hostnames)
            throws InterruptedException, ExecutionException {

        ExecutorService executor = Executors.newFixedThreadPool(20);
        Map<String, Future<List<String>>> futures = new LinkedHashMap<>();

        for (String hostname : hostnames) {
            futures.put(hostname, executor.submit(() -> {
                try {
                    return Arrays.stream(InetAddress.getAllByName(hostname))
                        .filter(a -> a instanceof Inet4Address)
                        .map(InetAddress::getHostAddress)
                        .collect(Collectors.toList());
                } catch (UnknownHostException e) {
                    return Collections.emptyList();
                }
            }));
        }

        executor.shutdown();
        executor.awaitTermination(30, TimeUnit.SECONDS);

        Map<String, List<String>> results = new LinkedHashMap<>();
        for (Map.Entry<String, Future<List<String>>> entry : futures.entrySet()) {
            results.put(entry.getKey(), entry.getValue().get());
        }
        return results;
    }
}
```

## Conclusion

Java's `InetAddress.getAllByName()` is the core of DNS resolution. Filter results with `instanceof Inet4Address` for IPv4-only output. `getCanonicalHostName()` triggers PTR (reverse) lookup. For production use, add a TTL-based in-memory cache to avoid redundant DNS queries, and use parallel resolution when processing many hostnames.
