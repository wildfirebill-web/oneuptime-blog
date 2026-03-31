# How to Design a Hierarchical Data Schema in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Hierarchical Data, Nested Set, Schema Design

Description: Compare adjacency list, nested sets, and closure table patterns for storing hierarchical data in MySQL with query examples.

---

Hierarchical data - categories, org charts, threaded comments, file paths - requires a deliberate storage strategy. MySQL offers several options, each with different query performance characteristics.

## Option 1 - Adjacency List

The simplest approach: each row stores its direct parent.

```sql
CREATE TABLE categories (
    id        INT UNSIGNED NOT NULL AUTO_INCREMENT,
    parent_id INT UNSIGNED NULL,
    name      VARCHAR(100) NOT NULL,
    PRIMARY KEY (id),
    KEY idx_parent (parent_id),
    CONSTRAINT fk_cat_parent FOREIGN KEY (parent_id)
        REFERENCES categories (id) ON DELETE CASCADE
);
```

Reads for a single level are fast, but fetching an entire subtree requires recursive CTEs (MySQL 8.0+) or multiple round trips.

## Option 2 - Nested Sets

Each node stores a `lft` and `rgt` value that encodes its position in a pre-order traversal.

```sql
CREATE TABLE nested_categories (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    lft  INT UNSIGNED NOT NULL,
    rgt  INT UNSIGNED NOT NULL,
    PRIMARY KEY (id),
    KEY idx_lft_rgt (lft, rgt)
);
```

Fetching all descendants is a single range query:

```sql
-- Get all descendants of the node with lft=2, rgt=9
SELECT * FROM nested_categories
WHERE lft BETWEEN 2 AND 9
ORDER BY lft;
```

The downside: inserts and moves require updating many rows to renumber `lft`/`rgt`.

## Option 3 - Closure Table (Recommended)

A separate table stores every ancestor-descendant pair. This gives fast reads and relatively simple writes.

```sql
CREATE TABLE nodes (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE node_paths (
    ancestor_id   INT UNSIGNED NOT NULL,
    descendant_id INT UNSIGNED NOT NULL,
    depth         INT UNSIGNED NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id),
    KEY idx_descendant (descendant_id),
    CONSTRAINT fk_path_ancestor   FOREIGN KEY (ancestor_id)   REFERENCES nodes (id) ON DELETE CASCADE,
    CONSTRAINT fk_path_descendant FOREIGN KEY (descendant_id) REFERENCES nodes (id) ON DELETE CASCADE
);
```

## Inserting a New Node

```sql
-- Insert the node itself
INSERT INTO nodes (name) VALUES ('Laptops');
SET @new_id = LAST_INSERT_ID();

-- Add self-reference
INSERT INTO node_paths (ancestor_id, descendant_id, depth)
VALUES (@new_id, @new_id, 0);

-- Copy ancestor paths from parent (parent_id = 1)
INSERT INTO node_paths (ancestor_id, descendant_id, depth)
SELECT ancestor_id, @new_id, depth + 1
FROM   node_paths WHERE descendant_id = 1;
```

## Querying All Descendants

```sql
SELECT n.name, p.depth
FROM   nodes n
JOIN   node_paths p ON p.descendant_id = n.id
WHERE  p.ancestor_id = 1
  AND  p.depth > 0
ORDER BY p.depth, n.name;
```

## Summary

Adjacency list is simplest to understand; use it with recursive CTEs for moderate tree sizes. Nested sets excel at read-heavy trees that rarely change. Closure tables offer the best balance of read and write performance for trees that change frequently. Choose based on your read-to-write ratio and the depth of the hierarchy.
