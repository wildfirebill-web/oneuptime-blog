# How to Use ClickHouse JDBC Driver in Java Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Java, JDBC, Database, Analytics

Description: Set up and use the official ClickHouse JDBC driver in Java applications to run queries, insert data, and manage connections.

---

## Adding the Dependency

Add the ClickHouse JDBC driver to your Maven project.

```xml
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.6.0</version>
</dependency>
```

For Gradle:

```bash
implementation 'com.clickhouse:clickhouse-jdbc:0.6.0'
```

## Opening a Connection

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.util.Properties;

Properties props = new Properties();
props.setProperty("user", "default");
props.setProperty("password", "");

String url = "jdbc:ch://localhost:8123/default";
Connection conn = DriverManager.getConnection(url, props);
```

The driver class is `com.clickhouse.jdbc.ClickHouseDriver` and is loaded automatically via the service loader.

## Running a Query

```java
import java.sql.ResultSet;
import java.sql.Statement;

try (Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery(
         "SELECT event, count() FROM events GROUP BY event ORDER BY count() DESC LIMIT 10")) {
    while (rs.next()) {
        String event = rs.getString(1);
        long count = rs.getLong(2);
        System.out.printf("%s: %d%n", event, count);
    }
}
```

## Parameterized Inserts

```java
import java.sql.PreparedStatement;

String sql = "INSERT INTO events (user_id, event, ts) VALUES (?, ?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setLong(1, 42L);
    ps.setString(2, "page_view");
    ps.setTimestamp(3, new java.sql.Timestamp(System.currentTimeMillis()));
    ps.executeUpdate();
}
```

## Batch Inserts

Use `addBatch` and `executeBatch` for high-throughput inserts.

```java
try (PreparedStatement ps = conn.prepareStatement(
        "INSERT INTO events (user_id, event) VALUES (?, ?)")) {
    for (int i = 0; i < 1000; i++) {
        ps.setLong(1, (long) i);
        ps.setString(2, "click");
        ps.addBatch();
    }
    ps.executeBatch();
}
```

## Configuring Connection Settings

Pass extra settings via URL query parameters.

```text
jdbc:ch://localhost:8123/default?compress=1&async_insert=1&wait_for_async_insert=0
```

Common settings:

| Parameter | Purpose |
|---|---|
| `compress` | Enable LZ4 compression |
| `async_insert` | Enable async insert on the server |
| `socket_timeout` | TCP socket timeout in milliseconds |

## Handling Exceptions

```java
try {
    stmt.executeQuery("SELECT 1");
} catch (java.sql.SQLException e) {
    System.err.println("ClickHouse error code: " + e.getErrorCode());
    System.err.println("SQL state: " + e.getSQLState());
}
```

## Summary

The ClickHouse JDBC driver follows the standard Java JDBC API, making it straightforward to integrate into any Java application. Use prepared statements for parameterized queries and `addBatch` for bulk inserts to maximize throughput.
