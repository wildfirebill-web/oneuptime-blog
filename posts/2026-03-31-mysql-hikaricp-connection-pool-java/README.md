# How to Use HikariCP Connection Pool for MySQL in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, HikariCP, Java, Connection Pool, Performance

Description: Learn how to configure HikariCP as the MySQL connection pool in Java, including optimal settings for throughput, timeout handling, and pool health monitoring.

---

## Introduction

HikariCP is widely regarded as the fastest JDBC connection pool for Java. It has a minimal footprint, near-zero overhead per connection acquisition, and sensible defaults that work well for most MySQL workloads. This guide provides a detailed configuration reference and usage patterns.

## Maven Dependency

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.2.0</version>
</dependency>
```

## Comprehensive HikariCP Configuration

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig config = new HikariConfig();

// JDBC connection
config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true");
config.setUsername("root");
config.setPassword("password");

// Pool sizing
config.setMaximumPoolSize(20);       // max total connections
config.setMinimumIdle(5);            // connections kept open when idle

// Timeout settings
config.setConnectionTimeout(30_000); // ms to wait for a connection from pool
config.setIdleTimeout(600_000);      // ms before idle connection is removed
config.setMaxLifetime(1_800_000);    // max lifetime of a connection (30 min)
config.setKeepaliveTime(60_000);     // keepalive ping interval

// Connection validation
config.setConnectionTestQuery("SELECT 1");

// Naming
config.setPoolName("MySQL-HikariPool");

// Performance properties for MySQL
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
config.addDataSourceProperty("useServerPrepStmts", "true");
config.addDataSourceProperty("useLocalSessionState", "true");
config.addDataSourceProperty("cacheResultSetMetadata", "true");
config.addDataSourceProperty("cacheServerConfiguration", "true");
config.addDataSourceProperty("elideSetAutoCommits", "true");
config.addDataSourceProperty("maintainTimeStats", "false");

HikariDataSource dataSource = new HikariDataSource(config);
```

## Using the DataSource

```java
import java.sql.*;
import java.util.*;

public class ProductRepository {
    private final HikariDataSource ds;

    public ProductRepository(HikariDataSource ds) {
        this.ds = ds;
    }

    public List<Map<String, Object>> findByMaxPrice(double maxPrice) throws SQLException {
        List<Map<String, Object>> results = new ArrayList<>();
        String sql = "SELECT id, name, price, stock FROM products WHERE price <= ? AND stock > 0";

        try (Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setDouble(1, maxPrice);
            try (ResultSet rs = ps.executeQuery()) {
                ResultSetMetaData meta = rs.getMetaData();
                int cols = meta.getColumnCount();
                while (rs.next()) {
                    Map<String, Object> row = new LinkedHashMap<>();
                    for (int i = 1; i <= cols; i++) {
                        row.put(meta.getColumnLabel(i), rs.getObject(i));
                    }
                    results.add(row);
                }
            }
        }
        return results;
    }
}
```

## Monitoring Pool Health with JMX

HikariCP exposes pool metrics via JMX:

```java
config.setRegisterMbeans(true);
```

Or programmatically via HikariPoolMXBean:

```java
HikariPoolMXBean poolBean = dataSource.getHikariPoolMXBean();

System.out.printf("Active connections: %d%n", poolBean.getActiveConnections());
System.out.printf("Idle connections:   %d%n", poolBean.getIdleConnections());
System.out.printf("Total connections:  %d%n", poolBean.getTotalConnections());
System.out.printf("Threads waiting:    %d%n", poolBean.getThreadsAwaitingConnection());
```

## Configuring via Properties File

```properties
# hikari.properties
dataSourceClassName=com.mysql.cj.jdbc.MysqlDataSource
dataSource.url=jdbc:mysql://localhost:3306/mydb
dataSource.user=root
dataSource.password=password
maximumPoolSize=20
minimumIdle=5
connectionTimeout=30000
idleTimeout=600000
maxLifetime=1800000
poolName=MySQL-HikariPool
```

```java
HikariConfig config = new HikariConfig("/hikari.properties");
HikariDataSource ds = new HikariDataSource(config);
```

## Summary

HikariCP achieves peak performance through bytecode instrumentation and lock-free algorithms. The critical configuration values are `maximumPoolSize` (set to 2x CPU cores for OLTP), `maxLifetime` (always less than MySQL's `wait_timeout`), and the MySQL-specific prepared statement cache properties. Enable `registerMbeans` for runtime visibility into pool utilization.
