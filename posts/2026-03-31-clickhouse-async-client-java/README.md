# How to Use ClickHouse Async Client in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Java, Async, Client, Performance

Description: Use the ClickHouse async Java client to send non-blocking queries and inserts, improving throughput in reactive and high-concurrency applications.

---

## Overview

The official `clickhouse-client` library supports asynchronous operations via `CompletableFuture`. This is useful when you want to issue multiple queries in parallel or integrate ClickHouse into a reactive pipeline.

## Dependency

```xml
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-client</artifactId>
    <version>0.6.0</version>
</dependency>
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-http-client</artifactId>
    <version>0.6.0</version>
</dependency>
```

## Creating a Client

```java
import com.clickhouse.client.*;

ClickHouseNode server = ClickHouseNode.of("http://localhost:8123/default");
ClickHouseClient client = ClickHouseClient.newInstance(ClickHouseProtocol.HTTP);
```

## Async Query

```java
CompletableFuture<ClickHouseResponse> future = client
    .read(server)
    .query("SELECT event, count() FROM events GROUP BY event LIMIT 10")
    .execute();

future.thenAccept(response -> {
    for (ClickHouseRecord record : response.records()) {
        String event = record.getValue(0).asString();
        long count = record.getValue(1).asLong();
        System.out.printf("%s: %d%n", event, count);
    }
    response.close();
}).exceptionally(ex -> {
    System.err.println("Query failed: " + ex.getMessage());
    return null;
});
```

## Running Multiple Queries in Parallel

```java
CompletableFuture<ClickHouseResponse> f1 = client
    .read(server)
    .query("SELECT count() FROM events WHERE toDate(ts) = today()")
    .execute();

CompletableFuture<ClickHouseResponse> f2 = client
    .read(server)
    .query("SELECT count() FROM errors WHERE toDate(ts) = today()")
    .execute();

CompletableFuture.allOf(f1, f2).join();

long eventCount = f1.get().firstRecord().getValue(0).asLong();
long errorCount = f2.get().firstRecord().getValue(0).asLong();
```

## Async Insert

```java
client.write(server)
    .table("metrics")
    .format(ClickHouseFormat.RowBinaryWithNamesAndTypes)
    .data(stream -> {
        // write rows to stream
    })
    .execute()
    .thenAccept(r -> System.out.println("Inserted: " + r.getSummary().getWrittenRows()));
```

## Timeout and Cancellation

```java
future.orTimeout(5, TimeUnit.SECONDS)
      .exceptionally(ex -> {
          System.err.println("Timed out");
          return null;
      });
```

## Closing the Client

```java
client.close();
```

The client is thread-safe and should be shared across the application, not created per request.

## Summary

The ClickHouse async Java client enables non-blocking queries and inserts using `CompletableFuture`. It is ideal for high-concurrency applications where blocking JDBC threads would create bottlenecks. Share one client instance, run parallel queries with `allOf`, and always close responses to release resources.
