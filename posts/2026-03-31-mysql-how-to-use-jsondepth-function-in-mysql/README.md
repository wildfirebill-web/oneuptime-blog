# How to Use JSON_DEPTH() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Json Depth, JSON, JSON Function, SQL

Description: Learn how to use MySQL's JSON_DEPTH() function to determine the maximum nesting depth of a JSON document for schema validation and complexity analysis.

---

## Overview

`JSON_DEPTH()` returns the maximum depth of a JSON document. A scalar value (string, number, boolean, null) has depth 1. An empty array or object also has depth 1. A non-empty array or object has depth at least 2. This function helps you detect overly nested JSON documents, validate schemas, and understand data complexity.

## Basic Syntax

```sql
JSON_DEPTH(json_doc)
```

Returns an integer representing the maximum nesting depth. Returns `NULL` if `json_doc` is `NULL`.

## Depth Definitions

```sql
-- Depth 1: scalars and empty containers
SELECT JSON_DEPTH('null');    -- 1
SELECT JSON_DEPTH('42');      -- 1
SELECT JSON_DEPTH('"hello"'); -- 1
SELECT JSON_DEPTH('true');    -- 1
SELECT JSON_DEPTH('{}');      -- 1
SELECT JSON_DEPTH('[]');      -- 1

-- Depth 2: flat object or array with scalar values
SELECT JSON_DEPTH('{"name":"Alice","age":30}'); -- 2
SELECT JSON_DEPTH('[1, 2, 3]');                 -- 2

-- Depth 3: one level of nesting inside an object or array
SELECT JSON_DEPTH('{"user":{"name":"Alice"}}'); -- 3
SELECT JSON_DEPTH('[[1, 2], [3, 4]]');          -- 3

-- Depth 4: two levels of nesting
SELECT JSON_DEPTH('{"a":{"b":{"c":1}}}');       -- 4
SELECT JSON_DEPTH('[[[1, 2]]]');                 -- 4
```

## Validating JSON Complexity

```sql
CREATE TABLE api_payloads (
  id INT AUTO_INCREMENT PRIMARY KEY,
  endpoint VARCHAR(200),
  payload JSON,
  received_at DATETIME DEFAULT NOW()
);

-- Find payloads that are too deeply nested (limit 5)
SELECT id, endpoint, JSON_DEPTH(payload) AS depth
FROM api_payloads
WHERE JSON_DEPTH(payload) > 5;

-- Reject overly complex payloads at insert time (application logic)
-- Or use a CHECK constraint (MySQL 8.0.16+)
ALTER TABLE api_payloads
  ADD CONSTRAINT chk_depth CHECK (JSON_DEPTH(payload) <= 10);
```

## Comparing Depth with Length

```sql
-- A wide document (many keys but shallow)
SELECT
  JSON_LENGTH('{"a":1,"b":2,"c":3,"d":4,"e":5}') AS length_val,  -- 5
  JSON_DEPTH('{"a":1,"b":2,"c":3,"d":4,"e":5}')  AS depth_val;   -- 2

-- A deep document (few keys but heavily nested)
SELECT
  JSON_LENGTH('{"a":{"b":{"c":{"d":1}}}}') AS length_val,   -- 1
  JSON_DEPTH('{"a":{"b":{"c":{"d":1}}}}')  AS depth_val;    -- 5
```

## Analyzing JSON Schema in a Column

```sql
-- Distribution of depths across all stored payloads
SELECT
  JSON_DEPTH(payload) AS depth,
  COUNT(*) AS count
FROM api_payloads
GROUP BY JSON_DEPTH(payload)
ORDER BY depth;

-- Average and max depth
SELECT
  AVG(JSON_DEPTH(payload)) AS avg_depth,
  MAX(JSON_DEPTH(payload)) AS max_depth
FROM api_payloads;
```

## Practical Example: Detecting Recursive Structures

Highly nested JSON (depth > 10) often indicates a bug or malicious input. You can use `JSON_DEPTH()` in a stored procedure to log or reject such payloads:

```sql
DELIMITER $$

CREATE PROCEDURE insert_api_payload(
  p_endpoint VARCHAR(200),
  p_payload  JSON
)
BEGIN
  IF JSON_DEPTH(p_payload) > 10 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'Payload nesting depth exceeds maximum allowed (10)';
  END IF;

  INSERT INTO api_payloads (endpoint, payload)
  VALUES (p_endpoint, p_payload);
END$$

DELIMITER ;
```

## JSON_DEPTH() vs JSON_LENGTH()

```sql
-- JSON_DEPTH: maximum nesting levels (how deep)
SELECT JSON_DEPTH('[[[1]]]');       -- 4

-- JSON_LENGTH: number of elements at the top level (how wide)
SELECT JSON_LENGTH('[[[1]],[2]]');  -- 2
```

## Summary

`JSON_DEPTH()` returns the maximum nesting depth of a JSON document, starting at 1 for scalars and empty containers. Use it to validate that incoming JSON does not exceed acceptable complexity thresholds, to analyze the structural characteristics of JSON columns, and to detect schema anomalies. Pair it with `JSON_LENGTH()` for a complete picture of both depth and breadth.
