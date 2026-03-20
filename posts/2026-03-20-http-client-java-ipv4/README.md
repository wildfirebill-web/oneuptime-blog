# How to Create an HTTP Client in Java That Connects via IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, HTTP, Client, IPv4, HttpClient, Networking

Description: Learn how to create an HTTP client in Java that connects via IPv4 using the modern java.net.http.HttpClient (Java 11+) and Apache HttpClient.

## Java 11+ HttpClient with IPv4

```java
import java.net.*;
import java.net.http.*;
import java.net.http.HttpResponse.BodyHandlers;
import java.time.Duration;

public class IPv4HttpClient {
    public static void main(String[] args) throws Exception {
        // Force IPv4 via system property (apply before creating the client)
        System.setProperty("java.net.preferIPv4Stack", "true");

        // Build the HttpClient with timeouts
        HttpClient client = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_1_1)  // or HTTP_2
            .connectTimeout(Duration.ofSeconds(10))
            .followRedirects(HttpClient.Redirect.NORMAL)
            .build();

        // Create a GET request
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.example.com/health"))
            .header("Accept", "application/json")
            .header("User-Agent", "MyApp/1.0")
            .timeout(Duration.ofSeconds(30))
            .GET()
            .build();

        // Send and get the response
        HttpResponse<String> response = client.send(request, BodyHandlers.ofString());

        System.out.println("Status: " + response.statusCode());
        System.out.println("Body: " + response.body());
        System.out.println("Headers: " + response.headers().map());
    }
}
```

## Forcing IPv4 via Custom DNS Resolution

```java
import java.net.*;
import java.net.http.*;
import java.net.http.HttpResponse.BodyHandlers;
import java.time.Duration;

public class IPv4OnlyHttpClient {
    // Custom Inet4Address-only DNS resolver
    private static URI resolveToIPv4(String originalUrl) throws Exception {
        URI uri = URI.create(originalUrl);
        String hostname = uri.getHost();

        InetAddress[] addresses = InetAddress.getAllByName(hostname);
        String ipv4 = null;
        for (InetAddress addr : addresses) {
            if (addr instanceof Inet4Address) {
                ipv4 = addr.getHostAddress();
                break;
            }
        }
        if (ipv4 == null) throw new UnknownHostException("No IPv4 for: " + hostname);

        // Replace hostname with IPv4 address in URI
        return new URI(uri.getScheme(), uri.getUserInfo(), ipv4, uri.getPort(),
            uri.getPath(), uri.getQuery(), uri.getFragment());
    }

    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();

        URI ipv4Uri = resolveToIPv4("https://api.example.com/data");

        HttpRequest request = HttpRequest.newBuilder()
            .uri(ipv4Uri)
            .header("Host", "api.example.com")  // Required for SNI with IP address
            .GET()
            .build();

        HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
        System.out.println("Status: " + response.statusCode());
    }
}
```

## POST Request with JSON

```java
import java.net.*;
import java.net.http.*;
import java.net.http.HttpResponse.BodyHandlers;
import java.time.Duration;

public class JsonPostExample {
    public static void main(String[] args) throws Exception {
        System.setProperty("java.net.preferIPv4Stack", "true");

        HttpClient client = HttpClient.newHttpClient();

        String jsonBody = "{\"username\":\"alice\",\"password\":\"secret\"}";

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.example.com/login"))
            .header("Content-Type", "application/json")
            .timeout(Duration.ofSeconds(30))
            .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
            .build();

        HttpResponse<String> response = client.send(request, BodyHandlers.ofString());

        System.out.println("Status: " + response.statusCode());
        System.out.println("Response: " + response.body());
    }
}
```

## Async HTTP Requests

```java
import java.net.http.*;
import java.net.http.HttpResponse.BodyHandlers;
import java.util.*;
import java.util.concurrent.*;

public class AsyncIPv4Client {
    public static void main(String[] args) throws Exception {
        System.setProperty("java.net.preferIPv4Stack", "true");

        HttpClient client = HttpClient.newHttpClient();

        List<String> urls = List.of(
            "https://api.example.com/users",
            "https://api.example.com/orders",
            "https://api.example.com/products"
        );

        // Launch all requests concurrently
        List<CompletableFuture<HttpResponse<String>>> futures = urls.stream()
            .map(url -> client.sendAsync(
                HttpRequest.newBuilder().uri(URI.create(url)).GET().build(),
                BodyHandlers.ofString()
            ))
            .toList();

        // Wait for all responses
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();

        for (var future : futures) {
            HttpResponse<String> resp = future.get();
            System.out.println(resp.uri() + " -> " + resp.statusCode());
        }
    }
}
```

## Conclusion

Java 11+'s `HttpClient` is the modern, built-in choice for HTTP requests. Force IPv4 with `java.net.preferIPv4Stack=true` or resolve hostnames explicitly to `Inet4Address` before building URIs. Use `sendAsync()` with `CompletableFuture` for concurrent requests. For older Java versions, Apache HttpClient provides similar functionality with explicit IPv4 configuration.
