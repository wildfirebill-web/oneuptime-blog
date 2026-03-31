# How to Return Multiple Result Sets from a MySQL Stored Procedure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Result Set, Programming, Application

Description: Learn how to return multiple result sets from a MySQL stored procedure and how to consume them in application code using various client libraries.

---

A MySQL stored procedure can execute multiple `SELECT` statements and return each as a separate result set to the calling application. This is useful for returning related data in a single round-trip - for example, fetching a customer record along with their recent orders and unpaid invoices.

## Returning Multiple Result Sets

Simply include multiple `SELECT` statements in the procedure body:

```sql
DELIMITER //

CREATE PROCEDURE get_customer_dashboard(IN p_customer_id INT)
BEGIN
    -- Result set 1: customer profile
    SELECT id, name, email, tier, created_at
    FROM customers
    WHERE id = p_customer_id;

    -- Result set 2: recent orders
    SELECT id, total, status, created_at
    FROM orders
    WHERE customer_id = p_customer_id
    ORDER BY created_at DESC
    LIMIT 10;

    -- Result set 3: unpaid invoices
    SELECT id, amount, due_date
    FROM invoices
    WHERE customer_id = p_customer_id
      AND paid = FALSE
    ORDER BY due_date;

    -- Result set 4: summary statistics
    SELECT
        COUNT(*) AS total_orders,
        SUM(total) AS lifetime_value,
        MAX(created_at) AS last_order_date
    FROM orders
    WHERE customer_id = p_customer_id
      AND status = 'delivered';
END //

DELIMITER ;
```

Call the procedure:

```sql
CALL get_customer_dashboard(42);
```

MySQL returns all four result sets in sequence.

## Consuming Multiple Result Sets in Python

Using `mysql-connector-python`:

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    user='app_user',
    password='secret',
    database='mydb'
)

cursor = conn.cursor()
cursor.callproc('get_customer_dashboard', [42])

result_sets = []
for result in cursor.stored_results():
    result_sets.append(result.fetchall())

customer_profile = result_sets[0]
recent_orders = result_sets[1]
unpaid_invoices = result_sets[2]
summary_stats = result_sets[3]

print("Customer:", customer_profile)
print("Orders:", recent_orders)

cursor.close()
conn.close()
```

## Consuming Multiple Result Sets in Node.js

Using the `mysql2` library:

```javascript
const mysql = require('mysql2/promise');

async function getCustomerDashboard(customerId) {
    const conn = await mysql.createConnection({
        host: 'localhost',
        user: 'app_user',
        password: 'secret',
        database: 'mydb',
        multipleStatements: true
    });

    const [results] = await conn.query(
        'CALL get_customer_dashboard(?)',
        [customerId]
    );

    // results is an array of result sets
    const customerProfile = results[0];
    const recentOrders = results[1];
    const unpaidInvoices = results[2];
    const summaryStats = results[3];

    await conn.end();
    return { customerProfile, recentOrders, unpaidInvoices, summaryStats };
}
```

## Consuming Multiple Result Sets in Java

Using JDBC:

```java
import java.sql.*;

public void getCustomerDashboard(int customerId) throws SQLException {
    String sql = "{CALL get_customer_dashboard(?)}";

    try (CallableStatement cs = conn.prepareCall(sql)) {
        cs.setInt(1, customerId);
        boolean hasResults = cs.execute();

        int resultSetIndex = 0;
        while (hasResults) {
            try (ResultSet rs = cs.getResultSet()) {
                System.out.println("Result set " + resultSetIndex++);
                ResultSetMetaData meta = rs.getMetaData();
                while (rs.next()) {
                    for (int i = 1; i <= meta.getColumnCount(); i++) {
                        System.out.print(meta.getColumnName(i) + ": "
                            + rs.getString(i) + "  ");
                    }
                    System.out.println();
                }
            }
            hasResults = cs.getMoreResults();
        }
    }
}
```

## Combining with OUT Parameters

A procedure can return both result sets and OUT parameters:

```sql
DELIMITER //

CREATE PROCEDURE get_orders_with_count(
    IN p_customer_id INT,
    OUT p_total_count INT
)
BEGIN
    SELECT COUNT(*) INTO p_total_count
    FROM orders WHERE customer_id = p_customer_id;

    SELECT id, total, status, created_at
    FROM orders
    WHERE customer_id = p_customer_id
    ORDER BY created_at DESC;
END //

DELIMITER ;

CALL get_orders_with_count(42, @count);
SELECT @count AS total_order_count;
```

## Summary

MySQL stored procedures return multiple result sets by executing multiple `SELECT` statements. Each `SELECT` produces a separate result set returned to the client in order. Most MySQL client libraries provide iteration APIs to consume them: `stored_results()` in Python's connector, array indexing in Node.js mysql2, and `getMoreResults()` in JDBC. This pattern reduces round-trips for dashboard-style queries that need several related data sets.
