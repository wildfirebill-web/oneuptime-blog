# How to Implement Nested Set Model in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Hierarchy, Pattern, Schema, Performance

Description: Learn how to implement the nested set model in MySQL for fast hierarchical reads, ideal for read-heavy category trees and menus.

---

## What Is the Nested Set Model?

The nested set model represents hierarchical data using left (`lft`) and right (`rgt`) boundary numbers. Each node's boundaries enclose all its descendants' boundaries. This allows retrieving entire subtrees with a single non-recursive range query - much faster than recursive CTEs for read-heavy workloads.

The trade-off: reads are fast, but inserts and moves require renumbering many rows.

## Schema Design

```sql
CREATE TABLE categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    lft INT NOT NULL,
    rgt INT NOT NULL,
    depth INT NOT NULL DEFAULT 0,
    INDEX idx_lft_rgt (lft, rgt),
    INDEX idx_rgt (rgt)
);
```

## Understanding the lft/rgt Values

Imagine a depth-first tree traversal that assigns an incrementing counter each time you enter or leave a node:

```text
Electronics (lft=1, rgt=10)
  Laptops (lft=2, rgt=7)
    Gaming Laptops (lft=3, rgt=4)
    Business Laptops (lft=5, rgt=6)
  Smartphones (lft=8, rgt=9)
```

```sql
INSERT INTO categories (id, name, lft, rgt, depth) VALUES
    (1, 'Electronics',     1, 10, 0),
    (2, 'Laptops',         2,  7, 1),
    (3, 'Gaming Laptops',  3,  4, 2),
    (4, 'Business Laptops',5,  6, 2),
    (5, 'Smartphones',     8,  9, 1);
```

## Reading the Tree (Fast Range Queries)

```sql
-- Get ALL descendants of 'Laptops' (id=2, lft=2, rgt=7)
SELECT name, depth
FROM categories
WHERE lft BETWEEN 2 AND 7
ORDER BY lft;
-- Returns: Laptops, Gaming Laptops, Business Laptops

-- Get ALL ancestors of 'Gaming Laptops' (id=3, lft=3, rgt=4)
SELECT name, depth
FROM categories
WHERE lft < 3 AND rgt > 4
ORDER BY lft;
-- Returns: Electronics, Laptops

-- Get the full tree indented
SELECT
    CONCAT(REPEAT('  ', depth), name) AS indented_name,
    lft, rgt, depth
FROM categories
ORDER BY lft;

-- Count descendants (without fetching them)
SELECT name, (rgt - lft - 1) / 2 AS descendant_count
FROM categories
WHERE id = 1;  -- Electronics has (10-1-1)/2 = 4 descendants
```

## Inserting a New Leaf Node

Inserting requires making room by incrementing existing lft/rgt values:

```sql
DELIMITER $$

CREATE PROCEDURE insert_category(
    IN p_name VARCHAR(100),
    IN p_parent_id INT
)
BEGIN
    DECLARE v_parent_rgt INT;
    DECLARE v_parent_depth INT;

    -- Get the parent's rgt and depth
    SELECT rgt, depth INTO v_parent_rgt, v_parent_depth
    FROM categories WHERE id = p_parent_id;

    -- Make room: shift all lft/rgt values >= parent's rgt by 2
    UPDATE categories SET rgt = rgt + 2 WHERE rgt >= v_parent_rgt;
    UPDATE categories SET lft = lft + 2 WHERE lft >= v_parent_rgt;

    -- Insert the new node at the newly created space
    INSERT INTO categories (name, lft, rgt, depth)
    VALUES (p_name, v_parent_rgt, v_parent_rgt + 1, v_parent_depth + 1);
END$$

DELIMITER ;

-- Add 'Ultrabooks' under 'Laptops' (id=2)
CALL insert_category('Ultrabooks', 2);
```

## Deleting a Node and Its Subtree

```sql
DELIMITER $$

CREATE PROCEDURE delete_category_tree(IN p_id INT)
BEGIN
    DECLARE v_lft INT;
    DECLARE v_rgt INT;
    DECLARE v_width INT;

    SELECT lft, rgt, rgt - lft + 1 INTO v_lft, v_rgt, v_width
    FROM categories WHERE id = p_id;

    -- Delete the node and all descendants
    DELETE FROM categories WHERE lft BETWEEN v_lft AND v_rgt;

    -- Close the gap
    UPDATE categories SET rgt = rgt - v_width WHERE rgt > v_rgt;
    UPDATE categories SET lft = lft - v_width WHERE lft > v_rgt;
END$$

DELIMITER ;

CALL delete_category_tree(2);  -- Deletes Laptops and all subcategories
```

## Checking If a Node Is an Ancestor

```sql
-- Is Electronics (id=1) an ancestor of Gaming Laptops (id=3)?
SELECT COUNT(*) AS is_ancestor
FROM categories ancestor
JOIN categories descendant ON descendant.lft BETWEEN ancestor.lft AND ancestor.rgt
WHERE ancestor.id = 1
  AND descendant.id = 3;
-- Returns 1 = yes
```

## When to Use Nested Set vs Adjacency List

Use nested set when:
- Reads vastly outnumber writes (product categories, menus, static taxonomies)
- You need fast subtree counts or ancestor lookups without recursion

Use adjacency list when:
- Frequent inserts and moves are required
- MySQL 8.0 recursive CTEs are available

## Summary

The nested set model stores hierarchy as left/right boundaries, enabling O(1) subtree retrieval with simple range queries. This makes it ideal for read-heavy hierarchies like website navigation menus and product category trees. The cost is complex insert and move logic that requires renumbering, so it suits stable hierarchies more than frequently-changing ones.
