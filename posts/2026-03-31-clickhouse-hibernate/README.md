# How to Use ClickHouse with Hibernate

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Hibernate, Java, ORM, Analytics

Description: Use Hibernate with ClickHouse in Java by leveraging native SQL queries and disabling unsupported DDL features for analytics workloads.

---

## Hibernate and ClickHouse Compatibility

Hibernate was built for OLTP databases with full ACID support. ClickHouse is an analytical engine with different semantics. The approach is to use Hibernate as a thin SQL-execution layer - rely on native queries and avoid entity lifecycle operations like `UPDATE` and `DELETE`.

## Maven Setup

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>6.4.4.Final</version>
</dependency>
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.6.0</version>
</dependency>
```

## hibernate.cfg.xml

```xml
<hibernate-configuration>
  <session-factory>
    <property name="hibernate.connection.driver_class">
      com.clickhouse.jdbc.ClickHouseDriver
    </property>
    <property name="hibernate.connection.url">
      jdbc:ch://localhost:8123/analytics
    </property>
    <property name="hibernate.connection.username">default</property>
    <property name="hibernate.connection.password"></property>
    <property name="hibernate.hbm2ddl.auto">none</property>
    <property name="hibernate.dialect">
      org.hibernate.dialect.H2Dialect
    </property>
  </session-factory>
</hibernate-configuration>
```

Set `hbm2ddl.auto=none` - let ClickHouse DDL be managed separately.

## Entity Definition

```java
import jakarta.persistence.*;

@Entity
@Table(name = "api_logs")
public class ApiLog {

    @Id
    @Column(name = "request_id")
    private String requestId;

    @Column(name = "endpoint")
    private String endpoint;

    @Column(name = "status_code")
    private int statusCode;

    @Column(name = "duration_ms")
    private long durationMs;

    // getters and setters
}
```

## Native SQL Queries

```java
Session session = sessionFactory.openSession();

List<Object[]> results = session.createNativeQuery(
    "SELECT endpoint, avg(duration_ms) AS avg_ms " +
    "FROM api_logs " +
    "WHERE toDate(ts) = today() " +
    "GROUP BY endpoint " +
    "ORDER BY avg_ms DESC",
    Object[].class
).list();

for (Object[] row : results) {
    System.out.printf("%s: %.2f ms%n", row[0], (Double) row[1]);
}
session.close();
```

## Avoiding Unsupported Operations

Do not use:
- `session.save` or `session.persist` for large volumes - prefer JDBC batch
- `session.update` or `session.delete` - ClickHouse mutations are asynchronous
- `@GeneratedValue(strategy = GenerationType.IDENTITY)` - ClickHouse has no auto-increment

## Inserting with Hibernate

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

ApiLog log = new ApiLog();
log.setRequestId(UUID.randomUUID().toString());
log.setEndpoint("/api/events");
log.setStatusCode(200);
log.setDurationMs(45L);

session.persist(log);
tx.commit();
session.close();
```

This works for low-volume inserts. Batch via JDBC for high throughput.

## Summary

Hibernate can be used with ClickHouse by disabling DDL automation and using native SQL queries for analytics. Avoid Hibernate's update and delete operations because ClickHouse handles mutations differently. This pattern gives you Hibernate's session management with full ClickHouse SQL power.
