# How to Configure IPv6 in Spring Boot Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, IPv6, Spring Boot, HTTP, REST API, Web

Description: Configure IPv6 support in Spring Boot applications including server binding, client IP extraction, request logging, and RestTemplate/WebClient usage.

## Binding Spring Boot to IPv6

```properties
# application.properties
server.address=::
server.port=8080
```

Or bind to a specific IPv6 address:

```properties
server.address=2001:db8::1
server.port=8080
```

For both IPv4 and IPv6 simultaneously on Spring Boot 3.x, configure the embedded Tomcat:

```java
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ServerConfig {

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> ipv6Customizer() {
        return factory -> {
            // [::] accepts both IPv4 (as mapped) and IPv6
            factory.setAddress(java.net.InetAddress.getByName("::"));
        };
    }
}
```

## Extracting Client IPv6 Address

```java
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.*;
import java.net.*;

@RestController
public class ClientIPController {

    @GetMapping("/my-ip")
    public String getClientIP(HttpServletRequest request) {
        String ip = getRemoteIP(request);
        return "Your IP: " + ip;
    }

    private String getRemoteIP(HttpServletRequest request) {
        // Check standard proxy headers
        String forwarded = request.getHeader("X-Forwarded-For");
        if (forwarded != null && !forwarded.isEmpty()) {
            return forwarded.split(",")[0].trim();
        }

        String realIP = request.getHeader("X-Real-IP");
        if (realIP != null && !realIP.isEmpty()) {
            return realIP;
        }

        String addr = request.getRemoteAddr();

        // Unwrap IPv4-mapped addresses from dual-stack
        try {
            InetAddress inet = InetAddress.getByName(addr);
            if (inet instanceof Inet6Address) {
                Inet4Address v4 = (Inet4Address) inet.getClass()
                    .getMethod("getIPv4Address").invoke(inet);
                if (v4 != null) return v4.getHostAddress();
            }
        } catch (Exception ignored) {}

        return addr;
    }
}
```

## IPv6 in Spring Security Allowed Addresses

```java
import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            // Allow health checks from IPv6 monitoring subnet
            .requestMatchers("/health").access(
                new org.springframework.security.web.access.expression
                    .DefaultHttpSecurityExpressionHandler()
                    .createSecurityExpressionRoot(null, null))
            // Restrict admin to loopback (IPv4 and IPv6)
            .requestMatchers("/admin/**")
                .hasIpAddress("127.0.0.1")
            .anyRequest().authenticated()
        );
        return http.build();
    }
}
```

## WebClient for IPv6 REST Calls

Spring WebClient (reactive) connects to IPv6 backends using bracket notation in URLs:

```java
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@Service
public class IPv6BackendClient {

    private final WebClient webClient;

    public IPv6BackendClient() {
        this.webClient = WebClient.builder()
            .baseUrl("http://[2001:db8::10]:8080")
            .defaultHeader("Content-Type", "application/json")
            .build();
    }

    public Mono<String> getHealth() {
        return webClient.get()
            .uri("/health")
            .retrieve()
            .bodyToMono(String.class);
    }

    public Mono<String> postData(String body) {
        return webClient.post()
            .uri("/data")
            .bodyValue(body)
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

## RestTemplate for IPv6 (Synchronous)

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.time.Duration;

@Service
public class IPv6RestClient {

    private final RestTemplate restTemplate;

    public IPv6RestClient(RestTemplateBuilder builder) {
        this.restTemplate = builder
            .connectTimeout(Duration.ofSeconds(5))
            .readTimeout(Duration.ofSeconds(10))
            .build();
    }

    public String getFromIPv6Service(String ipv6Addr, int port, String path) {
        // IPv6 URL requires brackets around the address
        String url = String.format("http://[%s]:%d%s", ipv6Addr, port, path);
        return restTemplate.getForObject(url, String.class);
    }
}
```

## Logging IPv6 Requests with Interceptor

```java
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.stereotype.Component;
import java.io.IOException;

@Component
public class IPv6LoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) request;
        String addr = httpReq.getRemoteAddr();
        String method = httpReq.getMethod();
        String uri = httpReq.getRequestURI();

        System.out.printf("[%s] %s %s%n", addr, method, uri);
        chain.doFilter(request, response);
    }
}
```

## Conclusion

Spring Boot supports IPv6 through the `server.address=::` property, which binds Tomcat/Netty to all IPv6 interfaces. `request.getRemoteAddr()` returns the client's IPv6 address (or IPv4-mapped on dual-stack). For outbound connections, WebClient and RestTemplate both accept IPv6 URLs with brackets. Spring Security's `hasIpAddress()` works with both IPv4 and IPv6 CIDR notation for IP-based access control.
