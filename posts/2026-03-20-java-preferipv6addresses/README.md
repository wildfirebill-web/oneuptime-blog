# How to Configure java.net.preferIPv6Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, IPv6, JVM, Configuration, Networking, DNS

Description: Configure Java's IPv6 address preference system properties to control whether JVM applications use IPv4 or IPv6 when both are available.

## The System Properties

Java provides two key system properties for controlling IP version preference:

| Property | Values | Effect |
|---|---|---|
| `java.net.preferIPv4Stack` | `true` / `false` (default) | `true` forces IPv4-only, disables IPv6 entirely |
| `java.net.preferIPv6Addresses` | `true` / `false` (default) / `system` | Controls address selection order |

The `preferIPv6Addresses` values:
- `false` (default): prefer IPv4 addresses in DNS results
- `true`: prefer IPv6 addresses in DNS results
- `system`: use OS address selection order (RFC 6724)

## Setting Properties from Command Line

```bash
# Prefer IPv6 when resolving hostnames

java -Djava.net.preferIPv6Addresses=true -jar app.jar

# Force IPv4 only (disables IPv6 entirely)
java -Djava.net.preferIPv4Stack=true -jar app.jar

# Use OS address selection (RFC 6724 compliant)
java -Djava.net.preferIPv6Addresses=system -jar app.jar
```

## Setting Properties in Code

```java
import java.net.InetAddress;

public class IPv6Preference {

    public static void setIPv6Preference(boolean preferV6) {
        if (preferV6) {
            System.setProperty("java.net.preferIPv6Addresses", "true");
        } else {
            System.setProperty("java.net.preferIPv6Addresses", "false");
        }
    }

    public static void demonstratePreference() throws Exception {
        // Without preference (defaults to IPv4 first)
        System.clearProperty("java.net.preferIPv6Addresses");
        InetAddress[] addrs = InetAddress.getAllByName("example.com");
        System.out.println("Default order:");
        for (InetAddress a : addrs) {
            System.out.println("  " + a.getHostAddress() + " (" +
                (a instanceof java.net.Inet6Address ? "IPv6" : "IPv4") + ")");
        }

        // With IPv6 preference
        System.setProperty("java.net.preferIPv6Addresses", "true");
        addrs = InetAddress.getAllByName("example.com");
        System.out.println("With preferIPv6Addresses=true:");
        for (InetAddress a : addrs) {
            System.out.println("  " + a.getHostAddress());
        }
    }

    public static void main(String[] args) throws Exception {
        demonstratePreference();
    }
}
```

## Effect on InetAddress.getByName()

```java
import java.net.*;

public class PreferenceEffect {

    public static void checkPreference(String hostname) throws UnknownHostException {
        InetAddress primary = InetAddress.getByName(hostname);
        System.out.printf("getByName(%s) with preferIPv6Addresses=%s → %s (%s)%n",
            hostname,
            System.getProperty("java.net.preferIPv6Addresses", "not set"),
            primary.getHostAddress(),
            primary instanceof Inet6Address ? "IPv6" : "IPv4"
        );
    }

    public static void main(String[] args) throws Exception {
        // Default: prefers IPv4
        checkPreference("example.com");

        // Set IPv6 preference
        System.setProperty("java.net.preferIPv6Addresses", "true");
        checkPreference("example.com");

        // System: let OS decide
        System.setProperty("java.net.preferIPv6Addresses", "system");
        checkPreference("example.com");
    }
}
```

## Spring Boot Configuration

In Spring Boot, configure IPv6 preference in `application.properties` and via JVM args:

```properties
# application.properties - for server binding
server.address=::
server.port=8080
```

```bash
# Docker / Kubernetes deployment with IPv6 preference
JAVA_OPTS="-Djava.net.preferIPv6Addresses=true"
java $JAVA_OPTS -jar app.jar
```

```java
// Programmatic check for IPv6 availability
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        // Set IPv6 preference before Spring context starts
        System.setProperty("java.net.preferIPv6Addresses", "true");
        SpringApplication.run(Application.class, args);
    }
}
```

## Checking Effective Preference

```java
import java.net.*;

public class CheckEffectivePreference {

    public static void printNetworkInfo() throws Exception {
        System.out.println("java.net.preferIPv4Stack: " +
            System.getProperty("java.net.preferIPv4Stack", "false"));
        System.out.println("java.net.preferIPv6Addresses: " +
            System.getProperty("java.net.preferIPv6Addresses", "false"));

        // Check what localhost resolves to
        InetAddress loopback = InetAddress.getLoopbackAddress();
        System.out.println("Loopback address: " + loopback.getHostAddress() +
            " (" + (loopback instanceof Inet6Address ? "IPv6" : "IPv4") + ")");

        // Check the default address for socket binding
        ServerSocket test = new ServerSocket(0);
        System.out.println("Default bind: " + test.getLocalSocketAddress());
        test.close();
    }

    public static void main(String[] args) throws Exception {
        printNetworkInfo();
    }
}
```

## Conclusion

`java.net.preferIPv6Addresses` controls DNS resolution order in Java applications. Set it to `true` to prefer IPv6 when both A and AAAA records exist. The `system` value defers to OS-level RFC 6724 address selection, which is the most standards-compliant option. `java.net.preferIPv4Stack=true` is the nuclear option - it completely disables IPv6 at the JVM level. For containerized applications, pass these as JVM system properties via `JAVA_OPTS` or `JAVA_TOOL_OPTIONS` environment variables.
