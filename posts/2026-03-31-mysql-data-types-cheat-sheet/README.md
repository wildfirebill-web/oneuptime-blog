# MySQL Data Types Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Type, Schema, Cheat Sheet

Description: A comprehensive MySQL data types cheat sheet covering numeric, string, date/time, JSON, and spatial types with storage sizes and usage guidance.

---

## Numeric Types

```text
TINYINT    - 1 byte  (-128 to 127) or (0 to 255 UNSIGNED)
SMALLINT   - 2 bytes (-32,768 to 32,767)
MEDIUMINT  - 3 bytes (-8M to 8M)
INT        - 4 bytes (-2.1B to 2.1B)
BIGINT     - 8 bytes (very large integers)
FLOAT      - 4 bytes (approximate, 7 decimal digits)
DOUBLE     - 8 bytes (approximate, 15 decimal digits)
DECIMAL(p,s)- exact numeric, p=precision, s=scale
BIT(n)     - n bits, 1-64
```

```sql
-- Exact monetary values - use DECIMAL
CREATE TABLE prices (
  amount DECIMAL(10, 2) NOT NULL
);

-- Auto-increment primary key
CREATE TABLE users (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY
);
```

## String Types

```text
CHAR(n)       - fixed-length, 0-255 chars
VARCHAR(n)    - variable-length, 0-65535 chars (row limit applies)
TINYTEXT      - up to 255 bytes
TEXT          - up to 65,535 bytes
MEDIUMTEXT    - up to 16 MB
LONGTEXT      - up to 4 GB
TINYBLOB      - up to 255 bytes binary
BLOB          - up to 65,535 bytes binary
MEDIUMBLOB    - up to 16 MB binary
LONGBLOB      - up to 4 GB binary
ENUM('a','b') - one value from a predefined list
SET('a','b')  - zero or more values from a predefined list
```

```sql
CREATE TABLE articles (
  title       VARCHAR(255) NOT NULL,
  body        LONGTEXT,
  status      ENUM('draft','published','archived') DEFAULT 'draft'
);
```

## Date and Time Types

```text
DATE          - YYYY-MM-DD
TIME          - HH:MM:SS
DATETIME      - YYYY-MM-DD HH:MM:SS (no time zone)
TIMESTAMP     - YYYY-MM-DD HH:MM:SS (UTC-stored, time-zone-aware)
YEAR          - 4-digit year
```

```sql
CREATE TABLE events (
  event_date  DATE NOT NULL,
  starts_at   DATETIME NOT NULL,
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## JSON Type (MySQL 5.7.8+)

```sql
CREATE TABLE configs (
  id     INT PRIMARY KEY,
  data   JSON
);

INSERT INTO configs VALUES (1, '{"theme":"dark","lang":"en"}');

-- Access JSON field
SELECT data->>'$.theme' FROM configs WHERE id = 1;
```

## Spatial Types

```text
GEOMETRY        - any spatial type
POINT           - a single coordinate
LINESTRING      - a sequence of points
POLYGON         - a closed shape
MULTIPOINT      - collection of points
MULTILINESTRING - collection of linestrings
MULTIPOLYGON    - collection of polygons
GEOMETRYCOLLECTION - mixed collection
```

```sql
CREATE TABLE locations (
  id    INT PRIMARY KEY,
  name  VARCHAR(100),
  coord POINT NOT NULL SRID 4326
);
```

## Type Selection Guidelines

```text
Primary keys        - INT or BIGINT UNSIGNED AUTO_INCREMENT
Short fixed strings - CHAR (e.g., ISO codes)
Variable text       - VARCHAR up to a few thousand chars
Long text content   - TEXT family
Currency/financials - DECIMAL (never FLOAT)
Timestamps          - TIMESTAMP (auto UTC) or DATETIME (explicit)
Structured metadata - JSON
Binary data/files   - BLOB (prefer object storage)
Status/categories   - ENUM for small fixed lists
```

## Summary

Choosing the right MySQL data type directly impacts storage efficiency, query performance, and data integrity. Use DECIMAL for money, BIGINT for large IDs, VARCHAR for text within a known bound, and TIMESTAMP for auto-managed UTC times. Avoid FLOAT for precision-sensitive data, and prefer JSON over serialized strings for structured semi-structured data.
