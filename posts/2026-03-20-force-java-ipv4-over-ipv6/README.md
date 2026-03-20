# How to Force Java to Use IPv4 Instead of IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, IPv4, IPv6, Dual-Stack, JVM, Networking, System Properties

Description: Learn how to configure the JVM to prefer or exclusively use IPv4 networking instead of IPv6 using system properties and programmatic approaches.

## The JVM Dual-Stack Behavior

By default, the JVM on dual-stack systems (where both IPv4 and IPv6 are available) prefers IPv6 for hostname resolution. This can cause issues when connecting to IPv4-only infrastructure.

## Method 1: JVM System Properties (Most Common)

```bash
# Force IPv4 by setting system property at JVM startup

java -Djava.net.preferIPv4Stack=true -jar myapp.jar

# OR prefer IPv4 address families for hostname resolution
java -Djava.net.preferIPv4Addresses=true -jar myapp.jar
```

## Difference Between the Two Properties

| Property | Effect |
|----------|--------|
| `java.net.preferIPv4Stack=true` | Disable IPv6 entirely; all sockets use IPv4 |
| `java.net.preferIPv4Addresses=true` | Prefer IPv4 addresses when both IPv4/IPv6 available |

Use `preferIPv4Stack=true` when you need guaranteed IPv4-only operation. Use `preferIPv4Addresses=true` when you want IPv4 preference but still allow IPv6 as fallback.

## Method 2: Set in Code (Before Any Network Operations)

```java
public class ForceIPv4 {
    static {
        // Must be set before any network classes are loaded
        System.setProperty("java.net.preferIPv4Stack", "true");
        System.setProperty("java.net.preferIPv4Addresses", "true");
    }

    public static void main(String[] args) throws Exception {
        // All network operations now use IPv4
        java.net.InetAddress addr = java.net.InetAddress.getByName("google.com");
        System.out.println("Resolved: " + addr.getHostAddress());
        System.out.println("Is IPv4: " + (addr instanceof java.net.Inet4Address));
    }
}
```

## Method 3: Explicit IPv4 Resolution

When you can't set global properties, resolve hostnames explicitly to IPv4:

```java
import java.net.*;

public static InetAddress getIPv4Only(String hostname) throws UnknownHostException {
    InetAddress[] addresses = InetAddress.getAllByName(hostname);
    for (InetAddress addr : addresses) {
        if (addr instanceof Inet4Address) {
            return addr;
        }
    }
    throw new UnknownHostException("No IPv4 address for: " + hostname);
}

// Use in socket creation
InetAddress ipv4Addr = getIPv4Only("api.example.com");
Socket socket = new Socket(ipv4Addr, 443);
System.out.println("Connected via: " + socket.getRemoteSocketAddress());
```

## Method 4: Maven/Gradle Surefire Plugin

```xml
<!-- pom.xml - Force IPv4 in tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>-Djava.net.preferIPv4Stack=true</argLine>
    </configuration>
</plugin>
```

```groovy
// build.gradle
test {
    jvmArgs '-Djava.net.preferIPv4Stack=true'
}
```

## Method 5: Spring Boot Configuration

```properties
# application.properties
java.net.preferIPv4Stack=true
```

Or in the main class:

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        System.setProperty("java.net.preferIPv4Stack", "true");
        SpringApplication.run(Application.class, args);
    }
}
```

## Verifying IPv4 Is Forced

```java
import java.net.*;

public class VerifyIPv4 {
    public static void main(String[] args) throws Exception {
        InetAddress addr = InetAddress.getByName("google.com");
        System.out.println("Resolved: " + addr.getHostAddress());
        System.out.println("Is IPv4: " + (addr instanceof Inet4Address));
        System.out.println("preferIPv4Stack: " +
            System.getProperty("java.net.preferIPv4Stack", "not set"));
    }
}
```

## Conclusion

The quickest way to force Java to use IPv4 is `java -Djava.net.preferIPv4Stack=true`. For code-level control, set the system property before any network operations or use explicit `Inet4Address` filtering. `preferIPv4Stack=true` disables IPv6 entirely, while `preferIPv4Addresses=true` keeps IPv6 available but prefers IPv4.
