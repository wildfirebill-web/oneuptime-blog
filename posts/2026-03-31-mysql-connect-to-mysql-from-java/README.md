# How to Connect to MySQL from Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Java, JDBC, Connection, Driver

Description: Learn how to connect to a MySQL database from Java using JDBC with the MySQL Connector/J driver, including connection pooling with HikariCP.

---

## Overview

Java connects to MySQL through JDBC (Java Database Connectivity). The official driver is MySQL Connector/J, published by Oracle. For production applications, wrap it with a connection pool such as HikariCP.

## Adding the Dependency

With Maven:

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>9.0.0</version>
</dependency>
```

With Gradle:

```groovy
implementation 'com.mysql:mysql-connector-j:9.0.0'
```

## Basic JDBC Connection

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class MySQLExample {
    public static void main(String[] args) throws SQLException {
        String url = "jdbc:mysql://localhost:3306/shop"
                   + "?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC";

        try (Connection conn = DriverManager.getConnection(url, "app_user", "secret")) {
            System.out.println("Connected: " + conn.getMetaData().getDatabaseProductVersion());

            String sql = "SELECT id, name, price FROM products WHERE price < ?";
            try (PreparedStatement ps = conn.prepareStatement(sql)) {
                ps.setDouble(1, 50.00);
                try (ResultSet rs = ps.executeQuery()) {
                    while (rs.next()) {
                        System.out.printf("%d  %s  %.2f%n",
                            rs.getInt("id"), rs.getString("name"), rs.getDouble("price"));
                    }
                }
            }
        }
    }
}
```

## Connection Pooling with HikariCP

Single `DriverManager.getConnection()` calls are too slow for web applications. Use HikariCP:

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/shop?serverTimezone=UTC");
config.setUsername(System.getenv("DB_USER"));
config.setPassword(System.getenv("DB_PASSWORD"));
config.setMaximumPoolSize(10);
config.setConnectionTimeout(30_000);
config.addDataSourceProperty("characterEncoding", "UTF-8");
config.addDataSourceProperty("useUnicode", "true");

HikariDataSource ds = new HikariDataSource(config);

// Borrow from pool
try (Connection conn = ds.getConnection()) {
    // use conn
}
```

## Inserting Data

```java
String insertSql = "INSERT INTO orders (customer_id, total) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(insertSql,
        PreparedStatement.RETURN_GENERATED_KEYS)) {
    ps.setInt(1, 42);
    ps.setDouble(2, 99.99);
    ps.executeUpdate();

    try (ResultSet keys = ps.getGeneratedKeys()) {
        if (keys.next()) {
            System.out.println("New order ID: " + keys.getLong(1));
        }
    }
}
```

## Using Transactions

```java
conn.setAutoCommit(false);
try {
    // ... execute statements
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
    throw e;
} finally {
    conn.setAutoCommit(true);
}
```

## Summary

Use MySQL Connector/J with JDBC for Java-MySQL connectivity. Always use `PreparedStatement` instead of string concatenation to prevent SQL injection. For production, use HikariCP for connection pooling and load credentials from environment variables. Add `characterEncoding=UTF-8&serverTimezone=UTC` to the JDBC URL to avoid encoding and timezone issues.
