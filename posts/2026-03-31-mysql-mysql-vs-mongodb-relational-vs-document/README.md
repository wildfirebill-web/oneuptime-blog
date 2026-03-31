# MySQL vs MongoDB: When to Choose Relational vs Document

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MongoDB, Database

Description: Compare MySQL and MongoDB to understand when a relational model beats a document store and vice versa, with practical examples and trade-off analysis.

---

MySQL and MongoDB represent two fundamentally different approaches to data storage: the relational model and the document model. Choosing between them depends on your data shape, query patterns, and scalability requirements.

## Data Model

MySQL organizes data into tables with fixed schemas. Relationships are expressed through foreign keys.

```sql
-- MySQL: normalized relational schema
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL
);

CREATE TABLE addresses (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT,
  street VARCHAR(255),
  city VARCHAR(100),
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

MongoDB stores JSON-like documents. Related data can be embedded or referenced.

```json
{
  "_id": "abc123",
  "email": "alice@example.com",
  "addresses": [
    { "street": "123 Main St", "city": "Austin" },
    { "street": "456 Oak Ave", "city": "Denver" }
  ]
}
```

## Schema Flexibility

MongoDB is schema-less - different documents in the same collection can have different fields. This makes it easy to iterate quickly.

MySQL enforces a schema. Adding a column requires an `ALTER TABLE`, which can be slow on large tables without proper tooling (pt-online-schema-change, gh-ost).

```sql
-- MySQL: adding a column to a large table
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- Use pt-osc or gh-ost for large tables to avoid locking
```

## Querying

MySQL uses SQL, a mature and widely understood language. Joins are first-class citizens.

```sql
-- MySQL: join users with their orders
SELECT u.email, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

MongoDB queries operate on documents. Joins exist (`$lookup`) but are less natural.

```javascript
// MongoDB aggregation pipeline
db.orders.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "user_id",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },
  { $group: { _id: "$user.email", order_count: { $sum: 1 } } }
]);
```

## Transactions

MySQL InnoDB supports full ACID multi-table transactions. MongoDB added multi-document transactions in version 4.0, but they carry more overhead and are generally less efficient than MySQL transactions.

```sql
-- MySQL multi-table transaction
BEGIN;
INSERT INTO orders (user_id, total) VALUES (1, 99.99);
UPDATE inventory SET stock = stock - 1 WHERE product_id = 42;
COMMIT;
```

## Scaling

MongoDB was built with horizontal scaling (sharding) as a first-class feature. MySQL can be sharded but it requires more effort - Vitess is a common solution.

For read scaling, both support replication with read replicas.

## When to Use Each

| Scenario | MySQL | MongoDB |
|---|---|---|
| Structured, relational data | Best fit | Possible but awkward |
| Flexible or nested documents | Harder | Best fit |
| Complex multi-table transactions | Excellent | Possible but costly |
| Rapid prototyping | Good | Excellent |
| Horizontal sharding | Needs Vitess | Native support |
| Reporting and analytics | Great with joins | Aggregation pipelines |

## Summary

Choose MySQL when your data is structured and relational, when you need strong ACID guarantees across multiple tables, or when you rely on complex joins and SQL reporting. Choose MongoDB when your data is document-oriented with variable structure, when you need built-in horizontal sharding, or when rapid schema iteration is a priority. Many applications use both - MySQL for transactional data and MongoDB for flexible content storage.
