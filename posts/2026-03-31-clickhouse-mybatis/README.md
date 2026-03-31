# How to Use ClickHouse with MyBatis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MyBatis, Java, SQL Mapper, Analytics

Description: Configure MyBatis to work with ClickHouse in a Java application, write mapper interfaces and XML SQL maps for analytics queries.

---

## Why MyBatis with ClickHouse

MyBatis gives you full SQL control without Hibernate's ORM overhead. For ClickHouse's analytical SQL (window functions, array functions, `FINAL` modifier), writing raw SQL in mapper XML is cleaner than fighting a JPQL translator.

## Maven Dependencies

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.6.0</version>
</dependency>
```

## DataSource Configuration

```text
spring.datasource.url=jdbc:ch://localhost:8123/analytics
spring.datasource.username=default
spring.datasource.password=
spring.datasource.driver-class-name=com.clickhouse.jdbc.ClickHouseDriver
mybatis.mapper-locations=classpath:mappers/*.xml
mybatis.configuration.map-underscore-to-camel-case=true
```

## Domain Class

```java
public class EventSummary {
    private String eventName;
    private long count;
    private String latestTs;
    // getters and setters
}
```

## Mapper Interface

```java
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import java.util.List;

@Mapper
public interface EventMapper {
    List<EventSummary> topEvents(@Param("days") int days);
    int insertEvent(Event event);
}
```

## XML Mapper File

`src/main/resources/mappers/EventMapper.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.EventMapper">

  <select id="topEvents" resultType="com.example.EventSummary">
    SELECT
      event_name   AS eventName,
      count()      AS count,
      max(ts)      AS latestTs
    FROM events
    WHERE ts >= now() - INTERVAL #{days} DAY
    GROUP BY event_name
    ORDER BY count DESC
    LIMIT 20
  </select>

  <insert id="insertEvent" parameterType="com.example.Event">
    INSERT INTO events (user_id, event_name, ts)
    VALUES (#{userId}, #{eventName}, #{ts})
  </insert>

</mapper>
```

## Calling the Mapper

```java
@Service
public class AnalyticsService {

    private final EventMapper eventMapper;

    public AnalyticsService(EventMapper eventMapper) {
        this.eventMapper = eventMapper;
    }

    public List<EventSummary> weeklyTopEvents() {
        return eventMapper.topEvents(7);
    }
}
```

## Batch Inserts with MyBatis

For bulk writes, use the `ExecutorType.BATCH` session.

```java
try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
    EventMapper mapper = session.getMapper(EventMapper.class);
    for (Event e : events) {
        mapper.insertEvent(e);
    }
    session.commit();
}
```

## Summary

MyBatis pairs well with ClickHouse because it lets you write idiomatic ClickHouse SQL directly in mapper XML files. Use standard mapper interfaces for queries and batch executor sessions for high-throughput inserts.
