# How to Use MySQL Testcontainers in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Testcontainers, Java, Integration Test

Description: Learn how to use the Testcontainers Java library to spin up a real MySQL container for integration tests, including schema setup, lifecycle management, and JUnit 5 integration.

---

## Why Testcontainers for MySQL Integration Tests

Testcontainers starts a real Docker container running MySQL before your tests and stops it after. This eliminates the need for a shared test database, ensures tests always run against the correct MySQL version, and prevents environment-specific failures.

## Dependencies

Add to `pom.xml`:

```xml
<dependencies>
  <!-- Testcontainers core and MySQL module -->
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.19.8</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <version>1.19.8</version>
    <scope>test</scope>
  </dependency>
  <!-- JUnit 5 Testcontainers extension -->
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.19.8</version>
    <scope>test</scope>
  </dependency>
  <!-- MySQL Connector/J -->
  <dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
  </dependency>
</dependencies>
```

## Basic Container Setup

```java
import org.junit.jupiter.api.*;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.sql.*;

@Testcontainers
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderRepositoryTest {

    @Container
    static final MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("app_test")
        .withUsername("testuser")
        .withPassword("testpass")
        .withInitScript("db/schema.sql");  // file in src/test/resources

    private static Connection connection;

    @BeforeAll
    static void setup() throws SQLException {
        connection = DriverManager.getConnection(
            mysql.getJdbcUrl(),
            mysql.getUsername(),
            mysql.getPassword()
        );
    }

    @AfterAll
    static void teardown() throws SQLException {
        if (connection != null) connection.close();
    }

    @BeforeEach
    void seedData() throws SQLException {
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("TRUNCATE TABLE orders");
            stmt.execute(
                "INSERT INTO orders (customer_id, total, status) VALUES (1, 99.99, 'PENDING')"
            );
        }
    }
}
```

Schema file at `src/test/resources/db/schema.sql`:

```sql
CREATE TABLE IF NOT EXISTS orders (
  id          BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT        NOT NULL,
  total       DECIMAL(10,2) NOT NULL,
  status      VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
  created_at  TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

## Writing Tests

```java
    @Test
    @Order(1)
    void testInsertOrder() throws SQLException {
        try (PreparedStatement ps = connection.prepareStatement(
            "INSERT INTO orders (customer_id, total, status) VALUES (?, ?, ?)",
            Statement.RETURN_GENERATED_KEYS)) {

            ps.setLong(1, 42L);
            ps.setBigDecimal(2, new java.math.BigDecimal("149.99"));
            ps.setString(3, "PENDING");
            ps.executeUpdate();

            ResultSet keys = ps.getGeneratedKeys();
            Assertions.assertTrue(keys.next());
            Assertions.assertTrue(keys.getLong(1) > 0);
        }
    }

    @Test
    @Order(2)
    void testQueryOrder() throws SQLException {
        try (PreparedStatement ps = connection.prepareStatement(
            "SELECT status FROM orders WHERE customer_id = ?")) {
            ps.setLong(1, 1L);
            ResultSet rs = ps.executeQuery();
            Assertions.assertTrue(rs.next());
            Assertions.assertEquals("PENDING", rs.getString("status"));
        }
    }
```

## Using with Spring Boot

When using Spring Boot, expose the container's JDBC URL as a `@DynamicPropertySource`:

```java
@SpringBootTest
@Testcontainers
class OrderServiceIntegrationTest {

    @Container
    static final MySQLContainer<?> mysql =
        new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("app_test")
            .withUsername("testuser")
            .withPassword("testpass");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private OrderService orderService;

    @Test
    void createOrderPersistsToDatabase() {
        Order order = orderService.create(1L, new java.math.BigDecimal("50.00"));
        Assertions.assertNotNull(order.getId());
    }
}
```

## Summary

Testcontainers for Java provides a clean, declarative way to run MySQL integration tests inside real Docker containers. Annotate the test class with `@Testcontainers`, declare a `@Container` field, and use `@DynamicPropertySource` to wire the container JDBC URL into Spring. Container lifecycle is managed automatically, ensuring a fresh MySQL instance for every test run.
