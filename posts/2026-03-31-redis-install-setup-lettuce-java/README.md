# How to Install and Set Up Lettuce for Redis in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, Lettuce, Maven

Description: Learn how to add the Lettuce Redis client to a Java project, establish a connection, and run basic commands with its async and sync APIs.

---

Lettuce is a scalable, thread-safe Redis client for Java built on Netty. Unlike Jedis, a single Lettuce connection can be shared across threads. It supports synchronous, asynchronous, and reactive APIs, making it the preferred client for modern Java applications and Spring Data Redis.

## Maven Dependency

```xml
<dependency>
  <groupId>io.lettuce</groupId>
  <artifactId>lettuce-core</artifactId>
  <version>6.3.2.RELEASE</version>
</dependency>
```

## Gradle Dependency

```groovy
dependencies {
    implementation 'io.lettuce:lettuce-core:6.3.2.RELEASE'
}
```

## Basic Connection (Sync API)

```java
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.sync.RedisCommands;

RedisClient client = RedisClient.create("redis://localhost:6379");
StatefulRedisConnection<String, String> connection = client.connect();
RedisCommands<String, String> commands = connection.sync();

commands.set("greeting", "Hello, Lettuce!");
String value = commands.get("greeting");
System.out.println(value); // Hello, Lettuce!

connection.close();
client.shutdown();
```

## Connecting with Authentication

```java
import io.lettuce.core.RedisURI;

RedisURI uri = RedisURI.builder()
    .withHost("localhost")
    .withPort(6379)
    .withAuthentication("default", "your-password".toCharArray())
    .build();

RedisClient client = RedisClient.create(uri);
StatefulRedisConnection<String, String> connection = client.connect();
```

## Connecting with TLS

```java
RedisURI uri = RedisURI.builder()
    .withHost("your-redis-host")
    .withPort(6380)
    .withSsl(true)
    .withAuthentication("default", "password".toCharArray())
    .build();
```

## Async API

Lettuce's async API returns `CompletableFuture`-like `RedisFuture` objects:

```java
import io.lettuce.core.api.async.RedisAsyncCommands;

RedisAsyncCommands<String, String> async = connection.async();

async.set("key", "value")
    .thenCompose(ok -> async.get("key"))
    .thenAccept(val -> System.out.println("Got: " + val));
```

## Reactive API

```java
import io.lettuce.core.api.reactive.RedisReactiveCommands;
import reactor.core.publisher.Mono;

RedisReactiveCommands<String, String> reactive = connection.reactive();

Mono<String> result = reactive.set("greeting", "Hello")
    .then(reactive.get("greeting"));

result.subscribe(System.out::println);
```

## Thread Safety

Lettuce connections are thread-safe. You can share one connection across the application:

```java
// Create once at startup - thread-safe
RedisClient client = RedisClient.create("redis://localhost:6379");
StatefulRedisConnection<String, String> sharedConnection = client.connect();

// Use synchronously from multiple threads
RedisCommands<String, String> sync = sharedConnection.sync();
```

## Ping and Info

```java
String pong = commands.ping();
System.out.println(pong); // PONG

String info = commands.info("server");
System.out.println(info);
```

## Shutdown

```java
connection.close();
client.shutdown();
```

## Summary

Lettuce is added via a single Maven or Gradle dependency and creates thread-safe connections over a single Netty channel. You choose from sync, async, or reactive command APIs on the same connection - unlike Jedis, which requires a pool for concurrent access. Lettuce is the recommended client for Spring applications and any reactive Java stack using Project Reactor or RxJava.
