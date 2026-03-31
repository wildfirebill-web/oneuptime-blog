# How to Use Redis Pipelining with Lettuce in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, Lettuce, Pipelining, Performance

Description: Learn how to use Redis pipelining with the Lettuce client in Java to batch commands, reduce latency, and maximize throughput for bulk operations.

---

Lettuce supports pipelining by flushing multiple commands in a single network write. Unlike Jedis where you open an explicit pipeline object, Lettuce handles this automatically in async and reactive modes - or explicitly via `setAutoFlushCommands(false)`.

## Automatic Pipelining (Async Mode)

When you send multiple async commands without awaiting each one, Lettuce batches them automatically:

```java
import io.lettuce.core.RedisClient;
import io.lettuce.core.RedisFuture;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.async.RedisAsyncCommands;

RedisClient client = RedisClient.create("redis://localhost:6379");
StatefulRedisConnection<String, String> connection = client.connect();
RedisAsyncCommands<String, String> async = connection.async();

// Fire multiple commands - Lettuce batches them automatically
List<RedisFuture<String>> futures = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
    futures.add(async.set("key:" + i, "value:" + i));
}

// Wait for all to complete
LettuceFutures.awaitAll(5, TimeUnit.SECONDS, futures.toArray(new RedisFuture[0]));
System.out.println("Done - all 1000 keys set");
```

## Explicit Pipelining with setAutoFlushCommands

For maximum control, disable auto-flush and flush manually:

```java
// Disable auto flush
connection.setAutoFlushCommands(false);
RedisAsyncCommands<String, String> async = connection.async();

List<RedisFuture<?>> futures = new ArrayList<>();

for (int i = 0; i < 500; i++) {
    futures.add(async.set("product:" + i, "item-" + i));
    futures.add(async.expire("product:" + i, 3600L));
}

// Flush all buffered commands at once
connection.flushCommands();

// Await all results
LettuceFutures.awaitAll(10, TimeUnit.SECONDS, futures.toArray(new RedisFuture[0]));

// Re-enable auto flush
connection.setAutoFlushCommands(true);
```

## Reading Results

```java
connection.setAutoFlushCommands(false);
RedisAsyncCommands<String, String> async = connection.async();

RedisFuture<String> f1 = async.get("user:1");
RedisFuture<String> f2 = async.get("user:2");
RedisFuture<Long> f3 = async.incr("visits");

connection.flushCommands();
LettuceFutures.awaitAll(5, TimeUnit.SECONDS, f1, f2, f3);

System.out.println("User 1: " + f1.get());
System.out.println("User 2: " + f2.get());
System.out.println("Visits: " + f3.get());

connection.setAutoFlushCommands(true);
```

## Reactive Pipelining

In reactive mode, commands are automatically batched:

```java
import io.lettuce.core.api.reactive.RedisReactiveCommands;
import reactor.core.publisher.Flux;

RedisReactiveCommands<String, String> reactive = connection.reactive();

Flux.range(1, 200)
    .flatMap(i -> reactive.set("item:" + i, "data-" + i))
    .then()
    .block();

System.out.println("200 items inserted reactively");
```

## Batch Loading with Chunks

For very large datasets, chunk into batches to avoid memory pressure:

```java
connection.setAutoFlushCommands(false);
RedisAsyncCommands<String, String> async = connection.async();

List<String[]> data = loadDataFromDatabase(); // your data source
int batchSize = 1000;

for (int i = 0; i < data.size(); i++) {
    async.hset("record:" + i, Map.of("field", data.get(i)[0]));
    if ((i + 1) % batchSize == 0) {
        connection.flushCommands();
        Thread.sleep(10); // brief pause between batches
    }
}
connection.flushCommands(); // flush remaining

connection.setAutoFlushCommands(true);
```

## Performance Comparison

```text
100k SET commands:

Without pipelining:  ~8,000 ms  (100k network round trips)
With pipelining:     ~400 ms    (commands batched)
Improvement:         ~20x faster
```

## Summary

Lettuce pipelining works differently than Jedis - async commands are automatically batched by Lettuce's Netty-based I/O. For explicit control, use `setAutoFlushCommands(false)`, queue commands, then call `flushCommands()` to send them all at once. Await results with `LettuceFutures.awaitAll()`. For reactive code, pipelining happens transparently when using `flatMap` on a `Flux`.
