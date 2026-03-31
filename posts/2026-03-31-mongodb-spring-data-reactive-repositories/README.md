# How to Use Spring Data MongoDB Reactive Repositories

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Spring, Java, Reactive, Repository

Description: Learn how to use Spring Data MongoDB reactive repositories with Project Reactor to build non-blocking MongoDB data access layers in reactive Spring applications.

---

Spring Data MongoDB provides reactive repository support built on Project Reactor. Instead of blocking threads while waiting for database responses, reactive repositories return `Mono` and `Flux` types that emit results asynchronously, making them ideal for high-concurrency applications.

## Adding the Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>
```

Configure the connection:

```text
spring.data.mongodb.uri=mongodb://localhost:27017/myapp
```

## Enabling Reactive MongoDB Repositories

```java
@SpringBootApplication
@EnableReactiveMongoRepositories
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## Defining the Document and Repository

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "events")
public class Event {
    @Id
    private String id;
    private String type;
    private String payload;
    private long timestamp;
}
```

Extend `ReactiveMongoRepository` for full reactive CRUD support:

```java
import org.springframework.data.mongodb.repository.ReactiveMongoRepository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface EventRepository extends ReactiveMongoRepository<Event, String> {

    Flux<Event> findByType(String type);
    Flux<Event> findByTimestampBetween(long start, long end);
    Mono<Long> countByType(String type);
    Mono<Void> deleteByType(String type);
}
```

## Performing CRUD Operations

All repository methods return reactive types:

```java
@Service
public class EventService {

    private final EventRepository repository;

    public EventService(EventRepository repository) {
        this.repository = repository;
    }

    public Mono<Event> createEvent(Event event) {
        return repository.save(event);
    }

    public Flux<Event> getEventsByType(String type) {
        return repository.findByType(type);
    }

    public Mono<Void> deleteAllByType(String type) {
        return repository.deleteByType(type);
    }
}
```

## Using in a WebFlux Controller

Wire reactive repositories into Spring WebFlux controllers for a fully non-blocking pipeline:

```java
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/events")
public class EventController {

    private final EventRepository repository;

    public EventController(EventRepository repository) {
        this.repository = repository;
    }

    @GetMapping
    public Flux<Event> listEvents() {
        return repository.findAll();
    }

    @GetMapping("/{id}")
    public Mono<Event> getEvent(@PathVariable String id) {
        return repository.findById(id)
            .switchIfEmpty(Mono.error(new RuntimeException("Event not found")));
    }

    @PostMapping
    @ResponseStatus(org.springframework.http.HttpStatus.CREATED)
    public Mono<Event> createEvent(@RequestBody Event event) {
        return repository.save(event);
    }
}
```

## Reactive Custom Queries with @Query

```java
import org.springframework.data.mongodb.repository.Query;

public interface EventRepository extends ReactiveMongoRepository<Event, String> {

    @Query("{ 'type': ?0, 'timestamp': { $gte: ?1 } }")
    Flux<Event> findByTypeAfter(String type, long since);
}
```

## Transforming Results with Reactor Operators

Reactive repositories compose naturally with Reactor operators:

```java
public Mono<EventSummary> getSummary(String type) {
    return repository.findByType(type)
        .count()
        .map(count -> new EventSummary(type, count));
}

public Flux<String> getPayloads(String type) {
    return repository.findByType(type)
        .map(Event::getPayload)
        .distinct();
}
```

## Summary

Spring Data MongoDB reactive repositories enable non-blocking database access using `Flux` and `Mono` return types. Extend `ReactiveMongoRepository`, define derived query methods returning reactive types, and compose the results with Reactor operators. When used with Spring WebFlux, the entire request pipeline stays non-blocking from HTTP request to database response, making this pattern ideal for high-throughput, low-latency services.
