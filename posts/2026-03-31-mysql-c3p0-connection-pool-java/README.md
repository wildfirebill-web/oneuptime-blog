# How to Use c3p0 Connection Pool for MySQL in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, c3p0, Java, Connection Pool, JDBC

Description: Learn how to configure c3p0 as a MySQL connection pool in Java, covering pool sizing, connection testing, and statement caching for reliable database access.

---

## Introduction

c3p0 is a mature, stable JDBC connection pool library that has been used in Java applications for over two decades. While HikariCP is faster in benchmarks, c3p0 is still widely used in legacy applications and provides robust connection testing and recovery features.

## Maven Dependency

```xml
<dependency>
    <groupId>com.mchange</groupId>
    <artifactId>c3p0</artifactId>
    <version>0.9.5.5</version>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.2.0</version>
</dependency>
```

## Creating and Configuring the Pool

```java
import com.mchange.v2.c3p0.ComboPooledDataSource;
import java.beans.PropertyVetoException;

public class C3P0DataSource {

    private static final ComboPooledDataSource cpds;

    static {
        try {
            cpds = new ComboPooledDataSource();
            cpds.setDriverClass("com.mysql.cj.jdbc.Driver");
            cpds.setJdbcUrl("jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=UTC");
            cpds.setUser("root");
            cpds.setPassword("password");

            // Pool sizing
            cpds.setInitialPoolSize(5);
            cpds.setMinPoolSize(5);
            cpds.setMaxPoolSize(20);
            cpds.setAcquireIncrement(3);  // connections acquired at a time when pool grows

            // Timeout settings
            cpds.setMaxIdleTime(300);          // seconds before idle connection is removed
            cpds.setMaxConnectionAge(1800);     // max connection age in seconds

            // Connection testing
            cpds.setTestConnectionOnCheckout(false); // impacts performance if true
            cpds.setTestConnectionOnCheckin(true);
            cpds.setIdleConnectionTestPeriod(60);    // test idle connections every 60s
            cpds.setPreferredTestQuery("SELECT 1");

            // Statement caching
            cpds.setMaxStatementsPerConnection(50);

            // Retry settings
            cpds.setAcquireRetryAttempts(3);
            cpds.setAcquireRetryDelay(1000);  // ms between retries

        } catch (PropertyVetoException e) {
            throw new RuntimeException("c3p0 configuration failed", e);
        }
    }

    public static ComboPooledDataSource getDataSource() {
        return cpds;
    }
}
```

## Performing Database Operations

```java
import java.sql.*;
import java.util.*;

public class ProductDAO {
    private final ComboPooledDataSource ds = C3P0DataSource.getDataSource();

    public List<String> findProductNames(double maxPrice) throws SQLException {
        List<String> names = new ArrayList<>();
        String sql = "SELECT name FROM products WHERE price <= ? AND stock > 0";

        try (Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setDouble(1, maxPrice);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    names.add(rs.getString("name"));
                }
            }
        }
        return names;
    }

    public void updateStock(int productId, int newStock) throws SQLException {
        try (Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "UPDATE products SET stock = ? WHERE id = ?")) {
            ps.setInt(1, newStock);
            ps.setInt(2, productId);
            ps.executeUpdate();
        }
    }
}
```

## Configuring via c3p0.properties

Place `c3p0.properties` in the classpath root:

```properties
c3p0.driverClass=com.mysql.cj.jdbc.Driver
c3p0.jdbcUrl=jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=UTC
c3p0.user=root
c3p0.password=password
c3p0.initialPoolSize=5
c3p0.minPoolSize=5
c3p0.maxPoolSize=20
c3p0.acquireIncrement=3
c3p0.maxIdleTime=300
c3p0.idleConnectionTestPeriod=60
c3p0.preferredTestQuery=SELECT 1
```

Then instantiate simply:

```java
ComboPooledDataSource cpds = new ComboPooledDataSource();
```

## Monitoring Pool Statistics

```java
import com.mchange.v2.c3p0.PooledDataSource;

PooledDataSource pds = (PooledDataSource) C3P0DataSource.getDataSource();
System.out.println("Num connections: " + pds.getNumConnectionsDefaultUser());
System.out.println("Busy connections: " + pds.getNumBusyConnectionsDefaultUser());
System.out.println("Idle connections: " + pds.getNumIdleConnectionsDefaultUser());
```

## Summary

c3p0 provides a reliable MySQL connection pool with built-in connection testing and automatic recovery from database outages. Set `idleConnectionTestPeriod` with `preferredTestQuery = "SELECT 1"` to detect broken connections proactively, and tune `maxPoolSize` to stay within MySQL's `max_connections`. For new projects, prefer HikariCP for lower latency; use c3p0 when maintaining existing applications that already depend on it.
