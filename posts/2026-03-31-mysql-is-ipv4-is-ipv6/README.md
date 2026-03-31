# How to Use IS_IPV4() and IS_IPV6() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Function, Network, Validation, IP Address

Description: Learn how to use IS_IPV4() and IS_IPV6() in MySQL to validate IP address strings before storing or processing network data.

---

## Overview

MySQL provides `IS_IPV4()` and `IS_IPV6()` as validation functions that return 1 if the argument is a valid IPv4 or IPv6 address string, and 0 otherwise. These functions are useful for data validation, filtering, and cleaning network-related data.

## IS_IPV4() Function

`IS_IPV4(expr)` returns 1 if the string is a valid dotted-decimal IPv4 address, 0 otherwise:

```sql
SELECT IS_IPV4('192.168.1.1');    -- Returns: 1
SELECT IS_IPV4('10.0.0.256');     -- Returns: 0 (octet > 255)
SELECT IS_IPV4('hello');          -- Returns: 0
SELECT IS_IPV4('2001:db8::1');    -- Returns: 0 (this is IPv6)
SELECT IS_IPV4(NULL);             -- Returns: NULL
```

## IS_IPV6() Function

`IS_IPV6(expr)` returns 1 if the string is a valid IPv6 address, 0 otherwise:

```sql
SELECT IS_IPV6('2001:db8::1');         -- Returns: 1
SELECT IS_IPV6('::1');                  -- Returns: 1 (loopback)
SELECT IS_IPV6('::ffff:192.168.1.1');  -- Returns: 1 (IPv4-mapped)
SELECT IS_IPV6('192.168.1.1');         -- Returns: 0 (this is IPv4)
SELECT IS_IPV6('not-valid');            -- Returns: 0
```

## Validating Data Before Insert

Use these functions in application queries to reject invalid IP addresses:

```sql
-- Only insert if the IP is valid IPv4 or IPv6
INSERT INTO connections (ip_address, connected_at)
SELECT INET6_ATON('203.0.113.5'), NOW()
WHERE IS_IPV4('203.0.113.5') = 1;
```

## Filtering Mixed Data

When a column stores raw strings that could be IPv4, IPv6, or invalid data, you can use these functions to categorize records:

```sql
SELECT
  raw_ip,
  CASE
    WHEN IS_IPV4(raw_ip) = 1 THEN 'IPv4'
    WHEN IS_IPV6(raw_ip) = 1 THEN 'IPv6'
    ELSE 'invalid'
  END AS ip_type
FROM raw_logs;
```

## Cleaning Bad Data

Use `IS_IPV4()` and `IS_IPV6()` to find and remove invalid IP address records:

```sql
-- Find rows with invalid IP addresses
SELECT id, raw_ip
FROM raw_logs
WHERE IS_IPV4(raw_ip) = 0
  AND IS_IPV6(raw_ip) = 0;

-- Delete invalid records
DELETE FROM raw_logs
WHERE IS_IPV4(raw_ip) = 0
  AND IS_IPV6(raw_ip) = 0;
```

## Using with INET6_ATON() Safely

Combining these validation functions with `INET6_ATON()` ensures safe conversion:

```sql
SELECT
  raw_ip,
  CASE
    WHEN IS_IPV4(raw_ip) = 1 THEN INET6_ATON(raw_ip)
    WHEN IS_IPV6(raw_ip) = 1 THEN INET6_ATON(raw_ip)
    ELSE NULL
  END AS binary_ip
FROM raw_logs;
```

## IS_IPV4_COMPAT() and IS_IPV4_MAPPED()

MySQL also provides two related functions for checking special IPv6 address forms:

```sql
-- IS_IPV4_COMPAT checks for IPv4-compatible IPv6 addresses (deprecated form)
SELECT IS_IPV4_COMPAT(INET6_ATON('::192.168.1.1'));  -- Returns: 1

-- IS_IPV4_MAPPED checks for IPv4-mapped IPv6 addresses
SELECT IS_IPV4_MAPPED(INET6_ATON('::ffff:192.168.1.1'));  -- Returns: 1
```

## Summary

`IS_IPV4()` and `IS_IPV6()` provide simple, reliable IP address validation directly in SQL. Use them to validate data before inserting, classify mixed IP data, and safely chain into conversion functions like `INET6_ATON()`. They complement the broader set of MySQL network functions and are essential for maintaining clean network data in your database.
