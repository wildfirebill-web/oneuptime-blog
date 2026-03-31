# How to Use Jedis Connection Pooling in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, Jedis, Connection Pool

Description: Learn how to configure Jedis connection pooling in Java for thread-safe, high-performance Redis access using JedisPool and JedisPooled.

---

A bare `Jedis` instance is not thread-safe - each thread needs its own connection. `JedisPool` solves this by maintaining a pool of reusable connections. Jedis 4+ also provides `JedisPooled`, which has an even simpler API.

## Basic JedisPool Setup

```java
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

JedisPoolConfig config = new JedisPoolConfig();
config.setMaxTotal(20);       // max active connections
config.setMaxIdle(10);        // max idle connections
config.setMinIdle(2);         // min idle connections kept alive
config.setTestOnBorrow(true); // validate connection before use

JedisPool pool = new JedisPool(config, "localhost", 6379);
```

## Borrowing and Returning Connections

Always use try-with-resources so connections are returned to the pool:

```java
try (Jedis jedis = pool.getResource()) {
    jedis.set("product:1", "Keyboard");
    String value = jedis.get("product:1");
    System.out.println(value);
} // connection automatically returned to pool
```

## JedisPooled (Jedis 4+)

`JedisPooled` wraps the pool internally and exposes Redis commands directly - no need to borrow/return manually:

```java
import redis.clients.jedis.JedisPooled;

JedisPooled jedis = new JedisPooled("localhost", 6379);

jedis.set("greeting", "hello");
String value = jedis.get("greeting");
System.out.println(value); // hello
```

This is the recommended API for new code.

## Authenticated Pool

```java
import redis.clients.jedis.JedisPoolConfig;
import redis.clients.jedis.JedisPool;

JedisPool pool = new JedisPool(
    new JedisPoolConfig(),
    "localhost",
    6379,
    2000,          // connection timeout (ms)
    "your-password"
);
```

With ACL username:

```java
JedisPool pool = new JedisPool(
    new JedisPoolConfig(),
    "localhost",
    6379,
    2000,          // timeout
    "username",
    "password"
);
```

## Pool Configuration Tuning

```java
JedisPoolConfig config = new JedisPoolConfig();

// Connection limits
config.setMaxTotal(50);
config.setMaxIdle(20);
config.setMinIdle(5);

// Wait behavior when pool is exhausted
config.setBlockWhenExhausted(true);
config.setMaxWait(Duration.ofSeconds(5));

// Health checks
config.setTestOnBorrow(true);
config.setTestOnReturn(false);
config.setTestWhileIdle(true);
config.setMinEvictableIdleDuration(Duration.ofMinutes(1));
config.setTimeBetweenEvictionRuns(Duration.ofSeconds(30));
```

## Singleton Pattern

```java
public class RedisConnectionManager {
    private static JedisPool pool;

    public static synchronized JedisPool getPool() {
        if (pool == null) {
            JedisPoolConfig config = new JedisPoolConfig();
            config.setMaxTotal(20);
            config.setMaxIdle(10);
            pool = new JedisPool(config, "localhost", 6379);
        }
        return pool;
    }

    public static void shutdown() {
        if (pool != null && !pool.isClosed()) {
            pool.close();
        }
    }
}
```

## Monitoring Pool Stats

```java
JedisPool pool = new JedisPool(config, "localhost", 6379);

// Check pool health
System.out.println("Active connections: " + pool.getNumActive());
System.out.println("Idle connections:   " + pool.getNumIdle());
System.out.println("Waiters:            " + pool.getNumWaiters());
```

## Closing the Pool

Close the pool when your application shuts down:

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    pool.close();
    System.out.println("Redis pool closed");
}));
```

## Summary

Jedis connection pooling is essential for multi-threaded Java applications. Use `JedisPooled` for the simplest approach - it manages pooling internally - or `JedisPool` when you need fine-grained control over pool configuration. Key settings to tune are `maxTotal` (concurrent connections), `maxWait` (wait time when pool is exhausted), and `testOnBorrow` (validate connections before use).
