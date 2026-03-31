# How to Build a ClickHouse Connection Pool with HikariCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, HikariCP, Java, Connection Pool, Performance

Description: Configure HikariCP to manage a ClickHouse JDBC connection pool for high-throughput Java analytics applications.

---

## Why Connection Pooling Matters

Opening a TCP connection to ClickHouse for every query adds latency. HikariCP maintains a pool of ready connections, reducing overhead for applications that issue many small queries or bursts of inserts.

## Maven Dependency

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.6.0</version>
</dependency>
```

## Basic HikariCP Configuration

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:ch://localhost:8123/default");
config.setUsername("default");
config.setPassword("");
config.setDriverClassName("com.clickhouse.jdbc.ClickHouseDriver");

// Pool size tuning
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setConnectionTimeout(30_000);  // 30 s
config.setIdleTimeout(600_000);       // 10 min
config.setMaxLifetime(1_800_000);     // 30 min

HikariDataSource dataSource = new HikariDataSource(config);
```

## Using the DataSource

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(
         "SELECT count() FROM events WHERE ts >= today() - 1")) {
    try (ResultSet rs = ps.executeQuery()) {
        if (rs.next()) {
            System.out.println("Events today: " + rs.getLong(1));
        }
    }
}
```

HikariCP returns the connection to the pool when the try-with-resources block exits.

## Tuning Pool Size for ClickHouse

ClickHouse is CPU-bound, not I/O bound. More connections than CPU cores rarely helps and can increase query queue depth. A reasonable starting point:

```text
maximumPoolSize = number_of_app_instances * cores_per_instance * 2
```

For a 4-core app server: `maximumPoolSize = 8`.

## Enabling Keep-Alive

ClickHouse closes idle TCP connections after `tcp_keep_alive_timeout` (default 290 s). Configure HikariCP to validate connections with a lightweight ping.

```java
config.setKeepaliveTime(60_000);  // ping every 60 s
config.setConnectionTestQuery("SELECT 1");
```

## Health Monitoring

Expose pool metrics via JMX or Micrometer.

```java
config.setMetricRegistry(meterRegistry); // Micrometer MeterRegistry
config.setRegisterMbeans(true);
```

Monitor `hikaricp.connections.active` and `hikaricp.connections.pending` to detect pool exhaustion.

## Summary

HikariCP is a battle-tested choice for pooling ClickHouse JDBC connections. Keep pool size proportional to server CPU, enable keep-alive to prevent stale connections, and expose metrics so you can detect saturation before users notice latency spikes.
