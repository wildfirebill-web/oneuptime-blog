# How to Create Hierarchical Dictionaries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, Hierarchical, Tree Structure, Data Modeling

Description: Learn how to create hierarchical dictionaries in ClickHouse to model parent-child relationships and traverse trees efficiently inside queries.

---

Hierarchical dictionaries in ClickHouse allow you to represent tree-like structures - such as organizational charts, product category trees, or geographic hierarchies - directly inside a dictionary. ClickHouse provides built-in functions to traverse the hierarchy without recursive CTEs.

## When to Use Hierarchical Dictionaries

Use hierarchical dictionaries when you need to:
- Roll up metrics to a parent category
- Find all ancestors of a node
- Check whether a node is a descendant of another node

## Source Table Structure

The source table needs a self-referencing parent key column:

```sql
CREATE TABLE categories
(
    id        UInt64,
    name      String,
    parent_id UInt64
)
ENGINE = MergeTree
ORDER BY id;

INSERT INTO categories VALUES
(1, 'Root',       0),
(2, 'Electronics',1),
(3, 'Phones',     2),
(4, 'Laptops',    2),
(5, 'Gaming',     1);
```

## Creating the Hierarchical Dictionary

```sql
CREATE DICTIONARY category_dict
(
    id        UInt64,
    name      String,
    parent_id UInt64 HIERARCHICAL
)
PRIMARY KEY id
SOURCE(CLICKHOUSE(
    HOST 'localhost' PORT 9000
    USER 'default' PASSWORD ''
    DB 'default' TABLE 'categories'
))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 600);
```

The `HIERARCHICAL` attribute marks the column that points to the parent record. The root node should have `parent_id = 0`.

## Traversing the Hierarchy

Get all ancestors of a node using `dictGetHierarchy`:

```sql
SELECT dictGetHierarchy('category_dict', toUInt64(3)) AS ancestors;
-- Returns: [3, 2, 1]
```

Check if a node is a descendant of another using `dictIsIn`:

```sql
SELECT dictIsIn('category_dict', toUInt64(3), toUInt64(1)) AS is_child_of_root;
-- Returns: 1
```

Get a node's name:

```sql
SELECT dictGet('category_dict', 'name', toUInt64(3)) AS category_name;
-- Returns: 'Phones'
```

## Roll-Up Example - Sales by Top-Level Category

```sql
SELECT
    dictGet('category_dict', 'name', dictGetHierarchy('category_dict', category_id)[2]) AS top_category,
    sum(revenue) AS total_revenue
FROM sales
WHERE event_date >= today() - 30
GROUP BY top_category
ORDER BY total_revenue DESC;
```

## Checking Dictionary Status

```sql
SELECT name, element_count, status
FROM system.dictionaries
WHERE name = 'category_dict';
```

## Limitations

- Only one hierarchy level attribute per dictionary is supported
- The hierarchy is stored as a flat hashed map; very deep trees can increase traversal cost
- Circular references should be avoided; ClickHouse does not detect them automatically

## Summary

Hierarchical dictionaries let you model parent-child trees inside ClickHouse and traverse them with `dictGetHierarchy` and `dictIsIn`. This enables rollup analytics, ancestry checks, and tree navigation without complex recursive queries, making category hierarchies and org charts first-class citizens in your analytics pipeline.
