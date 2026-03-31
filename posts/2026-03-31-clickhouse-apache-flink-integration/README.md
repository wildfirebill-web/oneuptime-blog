# How to Use Apache Flink with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Apache Flink, Flink, Streaming, Data Pipeline, JDBC

Description: Connect Apache Flink to ClickHouse using the Flink JDBC connector or the ClickHouse Flink connector for real-time stream processing.

---

Apache Flink is a powerful stream processing engine. Integrating Flink with ClickHouse allows you to process and enrich streaming data before loading it into ClickHouse for analytics.

## Option 1: Flink JDBC Connector

The simplest approach uses the Flink JDBC connector with the ClickHouse JDBC driver:

Add dependencies to your `pom.xml`:

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-jdbc</artifactId>
    <version>3.1.2-1.18</version>
</dependency>
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.6.0</version>
</dependency>
```

Write a Flink job that sinks to ClickHouse:

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Event> events = env
    .addSource(new KafkaSource<>(...))
    .map(EventParser::parse);

events.addSink(JdbcSink.sink(
    "INSERT INTO events (event_time, event_type, user_id) VALUES (?, ?, ?)",
    (statement, event) -> {
        statement.setTimestamp(1, Timestamp.from(event.getEventTime()));
        statement.setString(2, event.getEventType());
        statement.setInt(3, event.getUserId());
    },
    JdbcExecutionOptions.builder()
        .withBatchSize(5000)
        .withBatchIntervalMs(500)
        .build(),
    new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
        .withUrl("jdbc:clickhouse://clickhouse-host:8123/analytics")
        .withDriverName("com.clickhouse.jdbc.ClickHouseDriver")
        .withUsername("default")
        .withPassword("secret")
        .build()
));

env.execute("Events to ClickHouse");
```

## Option 2: ClickHouse Flink Connector

The official ClickHouse Flink connector supports async inserts and better performance:

```xml
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-flink</artifactId>
    <version>0.1.0</version>
</dependency>
```

```java
ClickHouseSink<Event> sink = ClickHouseSink.<Event>builder()
    .withDatabase("analytics")
    .withTable("events")
    .withHost("clickhouse-host")
    .withPort(8123)
    .withUsername("default")
    .withPassword("secret")
    .withBatchSize(10000)
    .build();

events.addSink(sink);
```

## Create the ClickHouse Target Table

```sql
CREATE TABLE events (
    event_time DateTime,
    event_type LowCardinality(String),
    user_id UInt32,
    enriched_data String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, event_time);
```

## Flink SQL with ClickHouse

Flink SQL also supports ClickHouse via the JDBC catalog:

```sql
CREATE TABLE clickhouse_events (
    event_time TIMESTAMP,
    event_type STRING,
    user_id INT
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:clickhouse://localhost:8123/analytics',
    'table-name' = 'events',
    'driver' = 'com.clickhouse.jdbc.ClickHouseDriver',
    'sink.buffer-flush.max-rows' = '5000',
    'sink.buffer-flush.interval' = '500ms'
);

INSERT INTO clickhouse_events
SELECT event_time, event_type, user_id FROM kafka_events;
```

## Handle Checkpointing

Enable Flink checkpointing to ensure at-least-once delivery:

```java
env.enableCheckpointing(60000);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30000);
```

## Summary

Apache Flink integrates with ClickHouse via the JDBC connector for simplicity or the official ClickHouse Flink connector for better performance. Use Flink's batch sink with buffer sizes of 5,000-10,000 rows to balance insert throughput and latency. Enable checkpointing for fault tolerance in production pipelines.
