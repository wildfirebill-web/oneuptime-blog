# How to Use GET_FORMAT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Date Formatting, Database

Description: Learn how to use MySQL GET_FORMAT() to retrieve predefined format strings for dates and times that can be passed to DATE_FORMAT() and STR_TO_DATE().

---

## What Is the GET_FORMAT() Function?

`GET_FORMAT()` returns a predefined format string for a given date/time type and locale standard. The returned string can be used directly as the format argument to `DATE_FORMAT()` or `STR_TO_DATE()`, making it a convenient way to apply standard locale-specific date formats without remembering format codes.

**Syntax:**

```sql
GET_FORMAT(type, standard)
```

- `type` - one of `DATE`, `DATETIME`, or `TIME`.
- `standard` - one of `'EUR'`, `'USA'`, `'JIS'`, `'ISO'`, or `'INTERNAL'`.
- Returns `NULL` if the combination is unsupported.

---

## Supported Format Strings

### DATE Formats

| Standard   | GET_FORMAT(DATE, ...)   | Example Output   |
|------------|-------------------------|------------------|
| `'EUR'`    | `'%d.%m.%Y'`            | `31.03.2026`     |
| `'USA'`    | `'%m.%d.%Y'`            | `03.31.2026`     |
| `'JIS'`    | `'%Y-%m-%d'`            | `2026-03-31`     |
| `'ISO'`    | `'%Y-%m-%d'`            | `2026-03-31`     |
| `'INTERNAL'`| `'%Y%m%d'`             | `20260331`       |

### DATETIME Formats

| Standard    | GET_FORMAT(DATETIME, ...) | Example Output            |
|-------------|---------------------------|---------------------------|
| `'EUR'`     | `'%Y-%m-%d %H.%i.%s'`    | `2026-03-31 14.30.45`     |
| `'USA'`     | `'%Y-%m-%d %H.%i.%s'`    | `2026-03-31 14.30.45`     |
| `'JIS'`     | `'%Y-%m-%d %H:%i:%s'`    | `2026-03-31 14:30:45`     |
| `'ISO'`     | `'%Y-%m-%d %H:%i:%s'`    | `2026-03-31 14:30:45`     |
| `'INTERNAL'`| `'%Y%m%d%H%i%s'`         | `20260331143045`          |

### TIME Formats

| Standard    | GET_FORMAT(TIME, ...)   | Example Output |
|-------------|-------------------------|----------------|
| `'EUR'`     | `'%H.%i.%s'`            | `14.30.45`     |
| `'USA'`     | `'%h:%i:%s %p'`         | `02:30:45 PM`  |
| `'JIS'`     | `'%H:%i:%s'`            | `14:30:45`     |
| `'ISO'`     | `'%H:%i:%s'`            | `14:30:45`     |
| `'INTERNAL'`| `'%H%i%s'`              | `143045`       |

---

## Basic Examples

```sql
SELECT GET_FORMAT(DATE, 'EUR');
-- Returns: '%d.%m.%Y'

SELECT GET_FORMAT(DATE, 'USA');
-- Returns: '%m.%d.%Y'

SELECT GET_FORMAT(DATE, 'INTERNAL');
-- Returns: '%Y%m%d'

SELECT GET_FORMAT(TIME, 'USA');
-- Returns: '%h:%i:%s %p'

SELECT GET_FORMAT(DATETIME, 'ISO');
-- Returns: '%Y-%m-%d %H:%i:%s'
```

---

## Using GET_FORMAT() with DATE_FORMAT()

```sql
-- Format today's date in European style
SELECT DATE_FORMAT(CURDATE(), GET_FORMAT(DATE, 'EUR'));
-- Returns: '31.03.2026'

-- Format current datetime in US style
SELECT DATE_FORMAT(NOW(), GET_FORMAT(DATETIME, 'USA'));
-- Returns: '2026-03-31 14.30.45'

-- Format time in 12-hour US format
SELECT DATE_FORMAT(NOW(), GET_FORMAT(TIME, 'USA'));
-- Returns: '02:30:45 PM'

-- Internal compact format (useful for filenames, keys)
SELECT DATE_FORMAT(NOW(), GET_FORMAT(DATE, 'INTERNAL'));
-- Returns: '20260331'
```

---

## How GET_FORMAT() Works

```mermaid
flowchart LR
    A["GET_FORMAT(type, standard)"] --> B[Look up format string table]
    B --> C[Return format string]
    C -->|Pass to DATE_FORMAT()| D[Formatted date/time string]
    C -->|Pass to STR_TO_DATE()| E[Parsed DATE or DATETIME value]
```

---

## Using GET_FORMAT() with STR_TO_DATE()

`GET_FORMAT()` is equally useful for parsing date strings in a known format:

```sql
-- Parse a European date string
SELECT STR_TO_DATE('31.03.2026', GET_FORMAT(DATE, 'EUR'));
-- Returns: '2026-03-31'

-- Parse a compact internal date
SELECT STR_TO_DATE('20260331', GET_FORMAT(DATE, 'INTERNAL'));
-- Returns: '2026-03-31'

-- Parse an ISO datetime string
SELECT STR_TO_DATE('2026-03-31 14:30:45', GET_FORMAT(DATETIME, 'ISO'));
-- Returns: '2026-03-31 14:30:45'
```

---

## Practical: Multi-Locale Date Display

```sql
CREATE TABLE invoices (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client VARCHAR(100),
    issue_date DATE,
    region VARCHAR(10)
);

INSERT INTO invoices (client, issue_date, region) VALUES
('ACME Corp', '2026-03-31', 'USA'),
('Euro GmbH', '2026-03-31', 'EUR'),
('Tokyo Inc', '2026-03-31', 'JIS');

-- Display date in region-appropriate format
SELECT
    client,
    region,
    DATE_FORMAT(issue_date, GET_FORMAT(DATE, region)) AS formatted_date
FROM invoices;
```

Result:

| client     | region | formatted_date |
|------------|--------|----------------|
| ACME Corp  | USA    | 03.31.2026     |
| Euro GmbH  | EUR    | 31.03.2026     |
| Tokyo Inc  | JIS    | 2026-03-31     |

---

## INTERNAL Format for Sorting and Keys

The `INTERNAL` format removes all separators, making dates sortable as plain strings and suitable for use in filenames or composite keys:

```sql
SELECT DATE_FORMAT(CURDATE(), GET_FORMAT(DATE, 'INTERNAL')) AS file_prefix;
-- Returns: '20260331'

-- Use as a partition key prefix
SELECT CONCAT(DATE_FORMAT(issue_date, GET_FORMAT(DATE, 'INTERNAL')), '-', id) AS unique_key
FROM invoices;
```

---

## Summary

`GET_FORMAT()` returns predefined format strings for `DATE`, `TIME`, and `DATETIME` values according to locale standards (`EUR`, `USA`, `JIS`, `ISO`, `INTERNAL`). Its main value is as an argument to `DATE_FORMAT()` for locale-aware display and to `STR_TO_DATE()` for locale-aware parsing. This avoids hardcoding format strings and makes multi-regional date handling cleaner and more maintainable. The `INTERNAL` format is particularly useful for creating compact, sortable date strings for keys and file names.
