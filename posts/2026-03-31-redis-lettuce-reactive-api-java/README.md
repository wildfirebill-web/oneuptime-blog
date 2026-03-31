# How to Use Lettuce Reactive API for Redis in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, Lettuce, Reactive, Project Reactor

Description: Learn how to use the Lettuce reactive API with Project Reactor to perform non-blocking Redis operations in Java with Mono and Flux types.

---

Lettuce provides a reactive API built on Project Reactor, giving you `Mono` and `Flux` types for non-blocking Redis operations. This is ideal for reactive Java applications using Spring WebFlux or Reactor-based microservices.

## Setup

```xml
<dependency>
  <groupId>io.lettuce</groupId>
  <artifactId>lettuce-core</artifactId>
  <version>6.3.2.RELEASE</version>
</dependency>
```

## Obtaining the Reactive API

```java
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.reactive.RedisReactiveCommands;

RedisClient client = RedisClient.create("redis://localhost:6379");
StatefulRedisConnection<String, String> connection = client.connect();
RedisReactiveCommands<String, String> reactive = connection.reactive();
```

## Basic Reactive Operations

```java
// SET then GET
reactive.set("name", "Alice")
    .then(reactive.get("name"))
    .subscribe(value -> System.out.println("Got: " + value));

// SET with TTL
reactive.setex("token", 3600, "abc123")
    .subscribe(ok -> System.out.println("Token stored: " + ok));
```

## Chaining with flatMap

```java
import reactor.core.publisher.Mono;

Mono<String> result = reactive.set("user:1:name", "Alice")
    .flatMap(ok -> reactive.get("user:1:name"))
    .map(name -> "Hello, " + name);

result.subscribe(System.out::println); // Hello, Alice
```

## Working with Flux (Multiple Values)

```java
import reactor.core.publisher.Flux;

// LRANGE returns a Flux
reactive.rpush("queue", "job1", "job2", "job3")
    .thenMany(reactive.lrange("queue", 0, -1))
    .subscribe(item -> System.out.println("Item: " + item));
```

## Reactive Hash Operations

```java
Map<String, String> fields = Map.of(
    "name", "Alice",
    "email", "alice@example.com",
    "role", "admin"
);

reactive.hset("user:1", fields)
    .thenMany(reactive.hgetall("user:1"))
    .subscribe(kv -> System.out.println(kv.getKey() + ": " + kv.getValue()));
```

## Reactive Sorted Set

```java
import io.lettuce.core.ScoredValue;

reactive.zadd("leaderboard", 100.0, "alice", 85.0, "bob", 95.0, "carol")
    .thenMany(reactive.zrangeWithScores("leaderboard", 0, -1))
    .subscribe(sv -> System.out.printf("%s -> %.1f%n", sv.getValue(), sv.getScore()));
```

## Error Handling

```java
reactive.get("possibly:missing:key")
    .switchIfEmpty(Mono.just("default_value"))
    .subscribe(System.out::println);

// With error handler
reactive.get("key")
    .doOnError(err -> System.err.println("Redis error: " + err.getMessage()))
    .onErrorReturn("fallback")
    .subscribe(System.out::println);
```

## Reactive Pipelining (Batch)

```java
import reactor.core.publisher.Flux;

Flux.range(1, 100)
    .flatMap(i -> reactive.set("item:" + i, "value:" + i))
    .then()
    .doOnSuccess(v -> System.out.println("All 100 items set"))
    .subscribe();
```

## Blocking to Get a Result (for Testing)

```java
String value = reactive.get("name").block();
System.out.println(value);
```

Avoid `.block()` in production reactive code - it defeats the purpose of non-blocking I/O.

## Cleanup

```java
connection.close();
client.shutdown();
```

## Summary

The Lettuce reactive API exposes Redis commands as `Mono` (single result) or `Flux` (stream of results) types from Project Reactor. Operations are non-blocking and composable via `flatMap`, `then`, and `thenMany`. This API integrates naturally with Spring WebFlux and Reactor pipelines, enabling fully reactive Redis interactions without thread blocking in high-concurrency Java applications.
