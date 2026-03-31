# How to Store IP Addresses in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, IP Address, Data Type

Description: Learn how to store IPv4 and IPv6 addresses in MySQL efficiently using INET_ATON, INET6_ATON, INT UNSIGNED, and VARBINARY columns.

---

Storing IP addresses as plain strings is convenient but wastes space and makes range queries slow. MySQL provides built-in functions to convert IP addresses to compact numeric formats, enabling efficient storage and fast subnet filtering.

## Storing IPv4 Addresses with INT UNSIGNED

An IPv4 address is a 32-bit number, which fits perfectly in an `INT UNSIGNED` column (4 bytes) compared to 15 bytes for a string like `192.168.100.200`.

```sql
CREATE TABLE access_logs (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  ip_address INT UNSIGNED NOT NULL,
  accessed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  INDEX idx_ip (ip_address)
);

-- Insert using INET_ATON to convert string to integer
INSERT INTO access_logs (ip_address)
VALUES (INET_ATON('192.168.1.100'));

-- Retrieve with INET_NTOA to convert back to string
SELECT INET_NTOA(ip_address) AS ip, accessed_at
FROM access_logs;
```

## Querying IP Ranges

Storing IPs as integers makes subnet range queries extremely fast because you can use simple `BETWEEN` comparisons:

```sql
-- Find all accesses from the 10.0.0.0/8 private range
SELECT INET_NTOA(ip_address) AS ip, accessed_at
FROM access_logs
WHERE ip_address BETWEEN INET_ATON('10.0.0.0')
                     AND INET_ATON('10.255.255.255');
```

This query uses the index on `ip_address` efficiently since it is a numeric range scan.

## Storing IPv6 Addresses with VARBINARY(16)

IPv6 addresses are 128 bits (16 bytes). Use `VARBINARY(16)` with `INET6_ATON()` and `INET6_NTOA()`:

```sql
CREATE TABLE ipv6_logs (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  ip_address VARBINARY(16) NOT NULL,
  logged_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  INDEX idx_ip6 (ip_address)
);

-- Insert IPv6 address
INSERT INTO ipv6_logs (ip_address)
VALUES (INET6_ATON('2001:db8::1'));

-- Retrieve
SELECT INET6_NTOA(ip_address) AS ip, logged_at
FROM ipv6_logs;
```

`INET6_ATON()` also handles IPv4 addresses, so you can use a single `VARBINARY(16)` column if you need to store both address families.

## Unified Dual-Stack Table

```sql
CREATE TABLE unified_logs (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  ip_address VARBINARY(16) NOT NULL,
  is_ipv6 TINYINT(1) NOT NULL DEFAULT 0,
  accessed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
);

-- Insert IPv4
INSERT INTO unified_logs (ip_address, is_ipv6)
VALUES (INET6_ATON('203.0.113.5'), 0);

-- Insert IPv6
INSERT INTO unified_logs (ip_address, is_ipv6)
VALUES (INET6_ATON('fe80::1'), 1);

-- Query all entries with their addresses
SELECT INET6_NTOA(ip_address) AS ip, is_ipv6, accessed_at
FROM unified_logs;
```

## Avoiding VARCHAR Storage

```sql
-- Avoid this: 15-45 bytes, no efficient range queries
CREATE TABLE bad_logs (
  ip_address VARCHAR(45)
);

-- Prefer this: 4 bytes, fast range queries
CREATE TABLE good_logs (
  ip_address INT UNSIGNED
);
```

## Summary

Store IPv4 addresses as `INT UNSIGNED` using `INET_ATON()` for insertion and `INET_NTOA()` for retrieval. For IPv6 or dual-stack tables, use `VARBINARY(16)` with `INET6_ATON()` and `INET6_NTOA()`. Both approaches are more compact than strings, support indexing, and enable fast subnet range queries with `BETWEEN` or comparison operators.
