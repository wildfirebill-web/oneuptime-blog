# How to Use INET_ATON() and INET_NTOA() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Function, Network, IP Address, Integer

Description: Learn how INET_ATON() and INET_NTOA() convert IPv4 addresses between dotted-decimal notation and integer form for efficient storage.

---

## Overview

MySQL provides `INET_ATON()` and `INET_NTOA()` for converting IPv4 addresses between their human-readable dotted-decimal form (e.g., `192.168.1.1`) and a compact unsigned integer representation. Storing IPs as integers rather than strings saves space and enables efficient range queries.

## INET_ATON() Function

`INET_ATON(expr)` converts a dotted-decimal IPv4 address string to an unsigned integer. Each octet is packed into a 32-bit number:

```sql
SELECT INET_ATON('192.168.1.1');
-- Returns: 3232235777

SELECT INET_ATON('10.0.0.1');
-- Returns: 167772161

SELECT INET_ATON('255.255.255.255');
-- Returns: 4294967295
```

The formula is: `a * 256^3 + b * 256^2 + c * 256 + d` where `a.b.c.d` is the IP.

## INET_NTOA() Function

`INET_NTOA(expr)` is the inverse - it converts an integer back to a dotted-decimal IPv4 string:

```sql
SELECT INET_NTOA(3232235777);
-- Returns: 192.168.1.1

SELECT INET_NTOA(167772161);
-- Returns: 10.0.0.1
```

## Storing IP Addresses Efficiently

Using `INT UNSIGNED` instead of `VARCHAR(15)` for IPv4 addresses reduces storage from up to 15 bytes to 4 bytes and allows fast range queries:

```sql
CREATE TABLE access_log (
  id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
  ip_address INT UNSIGNED NOT NULL,
  accessed_at DATETIME    NOT NULL,
  PRIMARY KEY (id),
  INDEX idx_ip (ip_address)
);

-- Insert with INET_ATON
INSERT INTO access_log (ip_address, accessed_at)
VALUES (INET_ATON('203.0.113.42'), NOW());

-- Read back with INET_NTOA
SELECT INET_NTOA(ip_address) AS ip, accessed_at
FROM access_log
LIMIT 10;
```

## Range Queries on IP Addresses

Integer storage makes CIDR-style range filtering straightforward:

```sql
-- Find all accesses from the 192.168.1.0/24 subnet
SELECT INET_NTOA(ip_address) AS ip, accessed_at
FROM access_log
WHERE ip_address BETWEEN INET_ATON('192.168.1.0')
                     AND INET_ATON('192.168.1.255');
```

## Handling NULL and Invalid Inputs

`INET_ATON()` returns `NULL` for invalid input strings:

```sql
SELECT INET_ATON('not-an-ip');
-- Returns: NULL

SELECT INET_ATON(NULL);
-- Returns: NULL
```

You should validate IP addresses in application code before passing them to `INET_ATON()`, or use `IS NOT NULL` checks in queries.

## Combining with Application Logic

A common pattern is to let the application layer call `INET_ATON()` at insert time and `INET_NTOA()` at read time, or to use generated columns for transparency:

```sql
CREATE TABLE visitors (
  id           INT UNSIGNED NOT NULL AUTO_INCREMENT,
  ip_int       INT UNSIGNED NOT NULL,
  ip_str       VARCHAR(15)  GENERATED ALWAYS AS (INET_NTOA(ip_int)) VIRTUAL,
  visited_at   DATETIME     NOT NULL,
  PRIMARY KEY (id)
);
```

## Summary

`INET_ATON()` and `INET_NTOA()` provide efficient IPv4 address storage by converting dotted-decimal strings to compact 32-bit integers and back. Storing IPs as `INT UNSIGNED` reduces storage overhead and enables fast range queries for subnet filtering, making these functions essential for network-related data in MySQL.
