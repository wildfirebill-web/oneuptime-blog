# How to Use MySQL with Java JDBC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Java, JDBC, Database Integration, Backend

Description: Learn how to connect to MySQL from Java using JDBC with prepared statements, result set handling, and transaction management examples.

---

## What is JDBC

JDBC (Java Database Connectivity) is Java's standard API for connecting to relational databases. For MySQL, you use the MySQL Connector/J driver which implements the JDBC specification.

## Adding the MySQL Connector/J Dependency

For Maven:

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
</dependency>
```

For Gradle:

```text
implementation 'com.mysql:mysql-connector-j:8.3.0'
```

## Establishing a Connection

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class DatabaseConnector {

    private static final String URL = "jdbc:mysql://localhost:3306/your_database?useSSL=false&serverTimezone=UTC";
    private static final String USER = "your_user";
    private static final String PASSWORD = "your_password";

    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USER, PASSWORD);
    }

    public static void main(String[] args) {
        try (Connection conn = getConnection()) {
            System.out.println("Connected: " + !conn.isClosed());
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

## Creating a Table

```java
try (Connection conn = getConnection();
     Statement stmt = conn.createStatement()) {

    stmt.execute("""
        CREATE TABLE IF NOT EXISTS employees (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            department VARCHAR(100),
            salary DECIMAL(10, 2)
        )
    """);
    System.out.println("Table created");
}
```

## Inserting with PreparedStatement

```java
String sql = "INSERT INTO employees (name, department, salary) VALUES (?, ?, ?)";

try (Connection conn = getConnection();
     PreparedStatement pstmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {

    pstmt.setString(1, "Alice Smith");
    pstmt.setString(2, "Engineering");
    pstmt.setDouble(3, 95000.00);
    pstmt.executeUpdate();

    try (ResultSet generatedKeys = pstmt.getGeneratedKeys()) {
        if (generatedKeys.next()) {
            System.out.println("Inserted ID: " + generatedKeys.getLong(1));
        }
    }
}
```

## Querying Data

```java
String sql = "SELECT id, name, department, salary FROM employees WHERE department = ?";

try (Connection conn = getConnection();
     PreparedStatement pstmt = conn.prepareStatement(sql)) {

    pstmt.setString(1, "Engineering");

    try (ResultSet rs = pstmt.executeQuery()) {
        while (rs.next()) {
            int id = rs.getInt("id");
            String name = rs.getString("name");
            double salary = rs.getDouble("salary");
            System.out.printf("ID: %d, Name: %s, Salary: %.2f%n", id, name, salary);
        }
    }
}
```

## Updating Records

```java
String sql = "UPDATE employees SET salary = ? WHERE id = ?";

try (Connection conn = getConnection();
     PreparedStatement pstmt = conn.prepareStatement(sql)) {

    pstmt.setDouble(1, 105000.00);
    pstmt.setInt(2, 1);
    int rowsUpdated = pstmt.executeUpdate();
    System.out.println("Rows updated: " + rowsUpdated);
}
```

## Deleting Records

```java
try (Connection conn = getConnection();
     PreparedStatement pstmt = conn.prepareStatement("DELETE FROM employees WHERE id = ?")) {

    pstmt.setInt(1, 5);
    int rowsDeleted = pstmt.executeUpdate();
    System.out.println("Rows deleted: " + rowsDeleted);
}
```

## Using Transactions

```java
Connection conn = null;
try {
    conn = getConnection();
    conn.setAutoCommit(false);

    PreparedStatement debit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance - ? WHERE id = ?");
    debit.setDouble(1, 500.00);
    debit.setInt(2, 1);
    debit.executeUpdate();

    PreparedStatement credit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?");
    credit.setDouble(1, 500.00);
    credit.setInt(2, 2);
    credit.executeUpdate();

    conn.commit();
    System.out.println("Transaction committed");
} catch (SQLException e) {
    if (conn != null) conn.rollback();
    throw e;
} finally {
    if (conn != null) conn.setAutoCommit(true);
    if (conn != null) conn.close();
}
```

## Using Connection Pooling with HikariCP

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/your_database");
config.setUsername("your_user");
config.setPassword("your_password");
config.setMaximumPoolSize(10);
config.setConnectionTimeout(30000);

HikariDataSource dataSource = new HikariDataSource(config);

try (Connection conn = dataSource.getConnection()) {
    // use connection
}
```

## Summary

Java JDBC with MySQL Connector/J enables robust database connectivity through `PreparedStatement` for parameterized queries, `ResultSet` for iterating results, and transaction management via `setAutoCommit(false)`. Use HikariCP for connection pooling in production, and always close connections, statements, and result sets with try-with-resources to prevent resource leaks.
