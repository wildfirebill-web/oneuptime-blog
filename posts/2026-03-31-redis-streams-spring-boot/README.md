# How to Use Redis Streams with Spring Boot

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Spring Boot, Stream, Event Sourcing, Java

Description: Process Redis Streams in Spring Boot using StreamMessageListenerContainer and consumer groups for reliable, at-least-once event delivery.

---

Redis Streams give you a persistent, ordered event log with consumer group support, making them a better choice than Pub/Sub when message delivery guarantees matter. Spring Data Redis provides a `StreamMessageListenerContainer` that handles reading, acknowledgment, and error recovery.

## Add Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

## Define the Event Model

```java
public record OrderEvent(String orderId, String status, double amount) {}
```

## Produce Events with RedisTemplate

```java
@Service
public class OrderProducer {

    private final StringRedisTemplate template;

    public OrderProducer(StringRedisTemplate template) {
        this.template = template;
    }

    public RecordId publish(String orderId, String status, double amount) {
        Map<String, String> fields = Map.of(
            "orderId", orderId,
            "status", status,
            "amount", String.valueOf(amount)
        );
        return template.opsForStream().add("orders:stream", fields);
    }
}
```

## Configure the Stream Listener Container

```java
@Configuration
public class StreamConfig {

    @Bean
    public StreamMessageListenerContainer<String, MapRecord<String, String, String>> listenerContainer(
            RedisConnectionFactory factory) {

        StreamMessageListenerContainer.StreamMessageListenerContainerOptions<String,
            MapRecord<String, String, String>> options =
            StreamMessageListenerContainer.StreamMessageListenerContainerOptions
                .builder()
                .pollTimeout(Duration.ofMillis(500))
                .build();

        return StreamMessageListenerContainer.create(factory, options);
    }
}
```

## Implement the Message Listener

```java
@Component
public class OrderStreamListener
        implements StreamListener<String, MapRecord<String, String, String>> {

    private static final Logger log = LoggerFactory.getLogger(OrderStreamListener.class);

    @Override
    public void onMessage(MapRecord<String, String, String> message) {
        Map<String, String> body = message.getValue();
        log.info("Processing order {}: status={}", body.get("orderId"), body.get("status"));
    }
}
```

## Register and Start the Listener

```java
@Configuration
public class StreamRegistration {

    @Bean
    public Subscription subscribe(
            StreamMessageListenerContainer<String, MapRecord<String, String, String>> container,
            OrderStreamListener listener,
            StringRedisTemplate template) {

        // Create consumer group if it doesn't exist
        try {
            template.opsForStream().createGroup("orders:stream", "order-group");
        } catch (Exception ignored) {}

        Subscription sub = container.receive(
            Consumer.from("order-group", "consumer-1"),
            StreamOffset.create("orders:stream", ReadOffset.lastConsumed()),
            listener
        );

        container.start();
        return sub;
    }
}
```

## Acknowledge Messages

For manual acknowledgment, use `RedisTemplate`:

```java
template.opsForStream().acknowledge("orders:stream", "order-group", message.getId());
```

## Inspect Pending Messages

```bash
redis-cli xpending orders:stream order-group - + 10
redis-cli xlen orders:stream
```

## Summary

Spring Data Redis `StreamMessageListenerContainer` automates the polling, dispatch, and error handling for Redis Streams consumers. Consumer groups distribute messages across instances, and message acknowledgment ensures at-least-once delivery. This makes Redis Streams a production-grade alternative to Kafka for lower-throughput event-driven Spring Boot applications.
