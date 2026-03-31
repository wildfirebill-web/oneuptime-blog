# How to Use ReactiveRedisTemplate in Spring Boot

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Spring Boot, Reactive, Java, WebFlux

Description: Use ReactiveRedisTemplate in Spring WebFlux applications to perform non-blocking Redis operations with Project Reactor Mono and Flux types.

---

In a Spring WebFlux application, blocking Redis calls defeat the purpose of reactive programming. `ReactiveRedisTemplate` provides a fully non-blocking API that returns `Mono` and `Flux` types, keeping your reactive pipeline intact.

## Add Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

## Configure ReactiveRedisTemplate

```java
@Configuration
public class RedisConfig {

    @Bean
    public ReactiveRedisTemplate<String, String> reactiveRedisTemplate(
            ReactiveRedisConnectionFactory factory) {
        RedisSerializationContext<String, String> ctx =
            RedisSerializationContext.<String, String>newSerializationContext(
                new StringRedisSerializer())
            .value(new StringRedisSerializer())
            .build();
        return new ReactiveRedisTemplate<>(factory, ctx);
    }
}
```

For JSON values, replace `StringRedisSerializer` with `Jackson2JsonRedisSerializer`.

## Basic String Operations

```java
@Service
public class CacheService {

    private final ReactiveRedisTemplate<String, String> template;
    private final ReactiveValueOperations<String, String> ops;

    public CacheService(ReactiveRedisTemplate<String, String> template) {
        this.template = template;
        this.ops = template.opsForValue();
    }

    public Mono<Boolean> set(String key, String value, Duration ttl) {
        return ops.set(key, value, ttl);
    }

    public Mono<String> get(String key) {
        return ops.get(key);
    }

    public Mono<Boolean> delete(String key) {
        return template.delete(key).map(count -> count > 0);
    }
}
```

## Use in a WebFlux Controller

```java
@RestController
@RequestMapping("/cache")
public class CacheController {

    private final CacheService cacheService;

    @GetMapping("/{key}")
    public Mono<ResponseEntity<String>> get(@PathVariable String key) {
        return cacheService.get(key)
            .map(value -> ResponseEntity.ok(value))
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @PutMapping("/{key}")
    public Mono<ResponseEntity<String>> set(
            @PathVariable String key, @RequestBody String value) {
        return cacheService.set(key, value, Duration.ofMinutes(10))
            .map(ok -> ResponseEntity.ok("Cached"));
    }
}
```

## Hash Operations

```java
public Mono<Boolean> hset(String key, String field, String value) {
    return template.opsForHash().put(key, field, value);
}

public Flux<Map.Entry<Object, Object>> hgetall(String key) {
    return template.opsForHash().entries(key);
}
```

## Reactive Pub/Sub

```java
public Flux<String> subscribe(String channel) {
    return template.listenToChannel(channel)
        .map(msg -> msg.getMessage());
}

public Mono<Long> publish(String channel, String message) {
    return template.convertAndSend(channel, message);
}
```

## Summary

`ReactiveRedisTemplate` integrates Redis into Spring WebFlux without introducing blocking calls. All operations return `Mono` or `Flux`, enabling proper back-pressure and composability in reactive pipelines. Configuring serialisers ensures clean JSON or string storage, and reactive Pub/Sub keeps the event-driven approach consistent throughout the application.
