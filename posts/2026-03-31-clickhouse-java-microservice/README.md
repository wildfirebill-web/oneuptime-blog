# How to Build a Java Microservice with ClickHouse Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Java, Microservice, Spring Boot, Analytics

Description: Build a Java Spring Boot microservice backed by ClickHouse to serve real-time analytics queries over an HTTP API.

---

## Architecture Overview

The microservice exposes a REST API. Each request triggers a ClickHouse query against a pre-aggregated analytics table. The service layer translates HTTP parameters into SQL, and the response serializes to JSON.

## Project Structure

```text
analytics-service/
  src/main/java/com/example/
    controller/AnalyticsController.java
    service/AnalyticsService.java
    repository/EventRepository.java
    model/EventSummary.java
  src/main/resources/
    application.properties
```

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.6.0</version>
</dependency>
```

## application.properties

```text
spring.datasource.url=jdbc:ch://localhost:8123/analytics
spring.datasource.username=default
spring.datasource.password=
spring.datasource.driver-class-name=com.clickhouse.jdbc.ClickHouseDriver
```

## Repository Layer

```java
@Repository
public class EventRepository {

    private final JdbcTemplate jdbc;

    public EventRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public List<EventSummary> topEvents(int days, int limit) {
        return jdbc.query(
            "SELECT event_name, count() AS cnt, avg(duration_ms) AS avg_ms " +
            "FROM events " +
            "WHERE ts >= now() - INTERVAL ? DAY " +
            "GROUP BY event_name ORDER BY cnt DESC LIMIT ?",
            (rs, row) -> new EventSummary(
                rs.getString("event_name"),
                rs.getLong("cnt"),
                rs.getDouble("avg_ms")
            ),
            days, limit
        );
    }
}
```

## Service Layer

```java
@Service
public class AnalyticsService {

    private final EventRepository repo;

    public AnalyticsService(EventRepository repo) {
        this.repo = repo;
    }

    public List<EventSummary> getTopEvents(int days) {
        if (days < 1 || days > 90) {
            throw new IllegalArgumentException("days must be 1-90");
        }
        return repo.topEvents(days, 20);
    }
}
```

## REST Controller

```java
@RestController
@RequestMapping("/api/analytics")
public class AnalyticsController {

    private final AnalyticsService service;

    public AnalyticsController(AnalyticsService service) {
        this.service = service;
    }

    @GetMapping("/top-events")
    public ResponseEntity<List<EventSummary>> topEvents(
            @RequestParam(defaultValue = "7") int days) {
        return ResponseEntity.ok(service.getTopEvents(days));
    }
}
```

## Caching Hot Queries

Wrap the service method with Spring Cache to avoid hitting ClickHouse on every request.

```java
@Cacheable(value = "topEvents", key = "#days")
public List<EventSummary> getTopEvents(int days) { ... }
```

Configure a 30-second TTL using Caffeine or Redis as the cache provider.

## Summary

A Spring Boot microservice backed by ClickHouse follows a standard layered architecture. Use `JdbcTemplate` for parameterized ClickHouse queries, validate inputs in the service layer, and add a short-lived cache to absorb repeated dashboard requests without increasing ClickHouse load.
