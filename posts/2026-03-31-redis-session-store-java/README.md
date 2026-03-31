# How to Build a Session Store in Java with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, Session

Description: Learn how to implement a Redis-backed session store in Java, covering creation, retrieval, expiration, and invalidation of user sessions.

---

Storing sessions in Redis instead of application memory allows horizontal scaling - any server in the cluster can read session data because it is all centralized in Redis. Sessions are stored as hashes keyed by a session token with TTL-based expiration.

## Session Service Implementation

```java
import redis.clients.jedis.UnifiedJedis;
import java.util.Map;
import java.util.UUID;
import java.util.HashMap;

public class SessionStore {
    private static final String SESSION_PREFIX = "session:";
    private static final int SESSION_TTL_SECONDS = 3600; // 1 hour

    private final UnifiedJedis jedis;

    public SessionStore(UnifiedJedis jedis) {
        this.jedis = jedis;
    }

    public String createSession(String userId, Map<String, String> attributes) {
        String token = UUID.randomUUID().toString();
        String key = SESSION_PREFIX + token;

        Map<String, String> sessionData = new HashMap<>(attributes);
        sessionData.put("userId", userId);
        sessionData.put("createdAt", String.valueOf(System.currentTimeMillis()));

        jedis.hset(key, sessionData);
        jedis.expire(key, SESSION_TTL_SECONDS);

        return token;
    }

    public Map<String, String> getSession(String token) {
        String key = SESSION_PREFIX + token;
        Map<String, String> session = jedis.hgetAll(key);
        return session.isEmpty() ? null : session;
    }

    public void refreshSession(String token) {
        jedis.expire(SESSION_PREFIX + token, SESSION_TTL_SECONDS);
    }

    public void setAttribute(String token, String field, String value) {
        jedis.hset(SESSION_PREFIX + token, field, value);
        jedis.expire(SESSION_PREFIX + token, SESSION_TTL_SECONDS);
    }

    public void invalidateSession(String token) {
        jedis.del(SESSION_PREFIX + token);
    }

    public boolean isValid(String token) {
        return jedis.exists(SESSION_PREFIX + token);
    }
}
```

## Servlet Filter for Session Validation

```java
@WebFilter("/*")
public class SessionFilter implements Filter {
    private final SessionStore sessionStore;

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String token = extractToken(request);

        if (token == null || !sessionStore.isValid(token)) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        // Refresh TTL on every valid request
        sessionStore.refreshSession(token);
        chain.doFilter(req, res);
    }

    private String extractToken(HttpServletRequest req) {
        Cookie[] cookies = req.getCookies();
        if (cookies == null) return null;
        for (Cookie c : cookies) {
            if ("SESSION".equals(c.getName())) return c.getValue();
        }
        return null;
    }
}
```

## Login and Logout Endpoints

```java
@RestController
public class AuthController {
    private final SessionStore sessionStore;

    @PostMapping("/login")
    public ResponseEntity<Map<String, String>> login(@RequestBody LoginRequest req,
                                                      HttpServletResponse response) {
        if (!authenticate(req.getUsername(), req.getPassword())) {
            return ResponseEntity.status(401).build();
        }

        Map<String, String> attrs = Map.of("username", req.getUsername(), "role", "user");
        String token = sessionStore.createSession("user:" + req.getUsername(), attrs);

        Cookie cookie = new Cookie("SESSION", token);
        cookie.setHttpOnly(true);
        cookie.setSecure(true);
        cookie.setPath("/");
        response.addCookie(cookie);

        return ResponseEntity.ok(Map.of("status", "ok"));
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@CookieValue("SESSION") String token) {
        sessionStore.invalidateSession(token);
        return ResponseEntity.noContent().build();
    }
}
```

## Spring Session with Redis (Simplified Alternative)

For Spring Boot applications, Spring Session handles all of this automatically:

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

```java
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 3600)
@Configuration
public class SessionConfig {}
```

## Summary

A Redis-backed session store uses hashes keyed by a UUID token with TTL-based expiration. Sessions survive server restarts and allow load balancers to route requests to any instance. For Spring Boot applications, Spring Session with Redis automates this entirely. For non-Spring apps, a service class with `hset`, `hgetAll`, `expire`, and `del` covers all session lifecycle operations.
