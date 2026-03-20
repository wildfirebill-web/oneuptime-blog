# How to Build Dual-Stack Applications in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, IPv6, Dual-Stack, Networking, TCP, Happy Eyeballs

Description: Build dual-stack Java applications that handle both IPv4 and IPv6 connections, implement Happy Eyeballs connection racing, and detect IP version at runtime.

## Dual-Stack Server with [::] Binding

On Linux, binding to `[::]` typically accepts both IPv4 and IPv6 (controlled by `net.ipv6.bindv6only`):

```java
import java.io.*;
import java.net.*;

public class DualStackServer {

    static String describeClient(InetAddress addr) {
        if (addr instanceof Inet6Address) {
            byte[] b = addr.getAddress();
            // Detect IPv4-mapped: ::ffff:x.x.x.x
            boolean mapped = true;
            for (int i = 0; i < 10; i++) {
                if (b[i] != 0) { mapped = false; break; }
            }
            if (mapped && (b[10] & 0xff) == 0xff && (b[11] & 0xff) == 0xff) {
                return String.format("IPv4-via-dual-stack (%d.%d.%d.%d)",
                    b[12] & 0xff, b[13] & 0xff, b[14] & 0xff, b[15] & 0xff);
            }
            return "IPv6 " + addr.getHostAddress();
        }
        return "IPv4 " + addr.getHostAddress();
    }

    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket();
        server.bind(new InetSocketAddress("::", 8080));
        System.out.println("Dual-stack server on " + server.getLocalSocketAddress());

        while (true) {
            Socket client = server.accept();
            System.out.println("Client: " + describeClient(client.getInetAddress()));
            client.close();
        }
    }
}
```

## Happy Eyeballs — Connecting to IPv6 First

Happy Eyeballs (RFC 8305) attempts IPv6 first, with IPv4 as fallback after a 250ms delay:

```java
import java.net.*;
import java.util.*;
import java.util.concurrent.*;

public class HappyEyeballs {

    public static Socket connect(String hostname, int port) throws Exception {
        // Resolve all addresses
        InetAddress[] all = InetAddress.getAllByName(hostname);

        List<InetAddress> v6 = new ArrayList<>();
        List<InetAddress> v4 = new ArrayList<>();

        for (InetAddress a : all) {
            if (a instanceof Inet6Address) v6.add(a);
            else v4.add(a);
        }

        // Try IPv6 first
        if (!v6.isEmpty()) {
            try {
                Socket s = new Socket();
                s.connect(new InetSocketAddress(v6.get(0), port), 3000);
                System.out.println("Connected via IPv6: " + v6.get(0).getHostAddress());
                return s;
            } catch (Exception e) {
                System.out.println("IPv6 failed, trying IPv4: " + e.getMessage());
            }
        }

        // Fall back to IPv4
        if (!v4.isEmpty()) {
            Socket s = new Socket();
            s.connect(new InetSocketAddress(v4.get(0), port), 3000);
            System.out.println("Connected via IPv4: " + v4.get(0).getHostAddress());
            return s;
        }

        throw new ConnectException("No address available for " + hostname);
    }

    public static void main(String[] args) throws Exception {
        try (Socket s = connect("example.com", 80)) {
            System.out.println("Connected: " + s.getRemoteSocketAddress());
        }
    }
}
```

## Detecting IP Version at Runtime

```java
import java.net.*;

public class IPVersionDetector {

    public static boolean hasIPv6Connectivity() {
        try {
            // Try to connect to a well-known IPv6 address
            Socket s = new Socket();
            s.connect(new InetSocketAddress("2001:4860:4860::8888", 53), 2000);
            s.close();
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    public static boolean hasIPv4Connectivity() {
        try {
            Socket s = new Socket();
            s.connect(new InetSocketAddress("8.8.8.8", 53), 2000);
            s.close();
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    public static String detectConnectivity() {
        boolean v4 = hasIPv4Connectivity();
        boolean v6 = hasIPv6Connectivity();

        if (v4 && v6) return "dual-stack";
        if (v6) return "IPv6-only";
        if (v4) return "IPv4-only";
        return "no connectivity";
    }

    public static void main(String[] args) {
        System.out.println("Connectivity: " + detectConnectivity());
    }
}
```

## Dual-Stack HTTP Client with HttpClient (Java 11+)

```java
import java.net.*;
import java.net.http.*;
import java.time.Duration;

public class DualStackHTTP {

    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_2)
            .connectTimeout(Duration.ofSeconds(10))
            // Java HttpClient uses system DNS which returns both A and AAAA records
            .build();

        // URL with IPv6 literal requires brackets
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("http://[2001:db8::1]:8080/health"))
            .GET()
            .build();

        HttpResponse<String> response = client.send(request,
            HttpResponse.BodyHandlers.ofString());

        System.out.println("Status: " + response.statusCode());
        System.out.println("Body: " + response.body());
    }
}
```

## Conclusion

Dual-stack Java applications work by binding servers to `[::]` which handles both IP versions. Detect IPv4-mapped addresses (`::ffff:x.x.x.x`) on dual-stack servers for accurate client IP logging. The Happy Eyeballs pattern improves connection time by preferring IPv6 with an IPv4 fallback. Java 11's `HttpClient` resolves hostnames through the system DNS, automatically using AAAA records when IPv6 is available. Use `java.net.preferIPv6Addresses=true` as a JVM system property to globally prefer IPv6 in DNS resolution order.
