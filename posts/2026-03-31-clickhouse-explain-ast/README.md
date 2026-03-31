# How to Use EXPLAIN AST in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, EXPLAIN, AST, Query Analysis

Description: Learn how to use EXPLAIN AST in ClickHouse to inspect the abstract syntax tree of a query for debugging and deep analysis.

---

`EXPLAIN AST` in ClickHouse prints the abstract syntax tree (AST) of a SQL statement as ClickHouse parses it. The AST is the structured, tree-shaped representation of your query before any semantic analysis or optimization takes place. It is most useful when you need to understand exactly how ClickHouse has parsed a complex expression, debug a query that produces unexpected results, or verify that aliases and subqueries are structured as intended.

## Basic EXPLAIN AST Syntax

```sql
EXPLAIN AST
SELECT user_id, count() AS cnt
FROM events
WHERE event_date = '2024-06-01'
GROUP BY user_id;
```

Sample output:

```text
SelectQuery
  select
    Identifier user_id
    Function count (alias cnt)
      arguments
        List
  tables
    TablesInSelectQueryElement
      TableExpression
        Identifier events
  where
    Function equals
      arguments
        List
          Identifier event_date
          Literal '2024-06-01'
  groupBy
    Identifier user_id
```

The tree mirrors the SQL structure: SELECT columns, FROM table, WHERE expression, and GROUP BY keys each appear as named child nodes.

## Understanding AST Node Types

### Identifier Nodes

Identifier nodes represent column or table references before alias resolution. They appear as plain names with no transformation.

```sql
EXPLAIN AST
SELECT t.user_id, t.amount
FROM transactions AS t;
```

```text
SelectQuery
  select
    Identifier t.user_id
    Identifier t.amount
  tables
    TablesInSelectQueryElement
      TableExpression
        Identifier transactions (alias t)
```

### Function Nodes

Function nodes hold the function name and a child `arguments` list. Aggregate functions, arithmetic, and string functions all appear the same way.

```sql
EXPLAIN AST
SELECT toStartOfDay(event_time) AS day, sum(value) AS total
FROM metrics;
```

```text
SelectQuery
  select
    Function toStartOfDay (alias day)
      arguments
        List
          Identifier event_time
    Function sum (alias total)
      arguments
        List
          Identifier value
  tables
    ...
```

### Subquery Nodes

Subqueries appear as nested `SelectQuery` nodes, making it easy to see how deeply they are nested.

```sql
EXPLAIN AST
SELECT user_id
FROM users
WHERE user_id IN (
    SELECT DISTINCT user_id
    FROM orders
    WHERE status = 'completed'
);
```

```text
SelectQuery
  select
    Identifier user_id
  tables
    TablesInSelectQueryElement
      TableExpression
        Identifier users
  where
    Function in
      arguments
        List
          Identifier user_id
          SelectQuery
            select
              Function distinct
                ...
            tables
              ...
            where
              Function equals
                ...
```

The inner `SelectQuery` is a direct child of the `in` function's argument list.

## Debugging Complex Expressions

### Alias Expansion Check

Use EXPLAIN AST to confirm that aliases are not prematurely expanded at parse time (ClickHouse preserves them in the AST):

```sql
EXPLAIN AST
SELECT
    event_date,
    count() AS cnt,
    cnt * 2 AS double_cnt
FROM events
GROUP BY event_date;
```

The AST will show `Identifier cnt` in the `double_cnt` expression, confirming the alias reference is preserved as-is at parse time. ClickHouse resolves it during semantic analysis, not during parsing.

### Verifying JOIN Condition Parsing

```sql
EXPLAIN AST
SELECT o.order_id, u.email
FROM orders AS o
INNER JOIN users AS u ON o.user_id = u.id
WHERE o.total > 100;
```

```text
SelectQuery
  select
    Identifier o.order_id
    Identifier u.email
  tables
    TablesInSelectQueryElement
      TableExpression
        Identifier orders (alias o)
    TablesInSelectQueryElement
      JoinExpression
        TableExpression
          Identifier users (alias u)
        JoinKind INNER
        JoinStrictness ALL
        Function equals
          arguments
            List
              Identifier o.user_id
              Identifier u.id
  where
    Function greater
      ...
```

You can confirm the join type, strictness, and ON condition are parsed correctly before running the query.

## Using EXPLAIN AST for DDL Statements

EXPLAIN AST works on DDL too, not just SELECT:

```sql
EXPLAIN AST
CREATE TABLE sensor_data
(
    sensor_id  UInt32,
    recorded_at DateTime,
    temperature Float32
)
ENGINE = MergeTree()
ORDER BY (sensor_id, recorded_at);
```

```text
CreateQuery sensor_data
  columns
    ColumnDeclaration sensor_id
      DataType UInt32
    ColumnDeclaration recorded_at
      DataType DateTime
    ColumnDeclaration temperature
      DataType Float32
  storage
    StorageAST MergeTree
  orderBy
    Identifier sensor_id
    Identifier recorded_at
```

This is useful for validating that your CREATE TABLE statement is parsed as expected before executing it in production.

## Summary

`EXPLAIN AST` shows the raw parsed representation of your SQL before any optimization or alias resolution. It is most valuable for debugging queries where you suspect a parsing issue, verifying join conditions and subquery structure, and inspecting DDL statements. Because the AST reflects the parser output directly, differences between what you wrote and what appears in the AST immediately highlight mismatches in how ClickHouse interprets your query.
