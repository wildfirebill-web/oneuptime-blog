# How to Use INET6_ATON() and INET6_NTOA() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Function, Network, IPv6, Binary

Description: Learn how INET6_ATON() and INET6_NTOA() convert IPv4 and IPv6 addresses to binary form and back for compact, efficient storage.

---

## Overview

MySQL's `INET6_ATON()` and `INET6_NTOA()` functions extend IP address handling to support both IPv4 and IPv6. They convert IP addresses to and from binary representations - `VARBINARY(4)` for IPv4 and `VARBINARY(16)` for IPv6 - making them ideal for modern dual-stack environments.

## INET6_ATON() Function

`INET6_ATON(addr)` converts an IPv4 or IPv6 address string to its binary form. For IPv4 it returns 4 bytes; for IPv6 it returns 16 bytes:

```sql
-- IPv4 address
SELECT HEX(INET6_ATON('192.168.1.1'));
-- Returns: C0A80101 (4 bytes)

-- IPv6 address
SELECT HEX(INET6_ATON('2001:db8::1'));
-- Returns: 20010DB8000000000000000000000001 (16 bytes)

-- IPv4-mapped IPv6 address
SELECT HEX(INET6_ATON('::ffff:192.168.1.1'));
```

## INET6_NTOA() Function

`INET6_NTOA(addr)` converts the binary representation back to a human-readable string:

```sql
-- Convert binary back to IPv6 string
SELECT INET6_NTOA(INET6_ATON('2001:db8::1'));
-- Returns: 2001:db8::1

-- Convert binary back to IPv4 string
SELECT INET6_NTOA(INET6_ATON('192.168.1.1'));
-- Returns: 192.168.1.1
```

## Storing IPv4 and IPv6 Addresses

The recommended column type for storing both IPv4 and IPv6 is `VARBINARY(16)`:

```sql
CREATE TABLE connections (
  id         BIGINT       NOT NULL AUTO_INCREMENT,
  ip_address VARBINARY(16) NOT NULL,
  port       SMALLINT     NOT NULL,
  connected_at DATETIME   NOT NULL,
  PRIMARY KEY (id),
  INDEX idx_ip (ip_address)
);

-- Insert IPv4
INSERT INTO connections (ip_address, port, connected_at)
VALUES (INET6_ATON('203.0.113.5'), 443, NOW());

-- Insert IPv6
INSERT INTO connections (ip_address, port, connected_at)
VALUES (INET6_ATON('2001:db8::cafe'), 443, NOW());

-- Read back
SELECT INET6_NTOA(ip_address) AS ip, port, connected_at
FROM connections;
```

## Differentiating IPv4 from IPv6

You can use `LENGTH()` on the binary column to distinguish IPv4 from IPv6:

```sql
SELECT
  INET6_NTOA(ip_address) AS ip,
  CASE LENGTH(ip_address)
    WHEN 4  THEN 'IPv4'
    WHEN 16 THEN 'IPv6'
    ELSE 'unknown'
  END AS ip_version
FROM connections;
```

## Range Queries for IPv6 Subnets

Binary storage allows range-based subnet queries for IPv6 just like IPv4:

```sql
-- All addresses in the 2001:db8::/32 block
SELECT INET6_NTOA(ip_address) AS ip
FROM connections
WHERE ip_address >= INET6_ATON('2001:db8::')
  AND ip_address <= INET6_ATON('2001:db8:ffff:ffff:ffff:ffff:ffff:ffff');
```

## Handling NULL and Invalid Input

Like `INET_ATON()`, `INET6_ATON()` returns `NULL` for invalid addresses:

```sql
SELECT INET6_ATON('not-valid');
-- Returns: NULL

SELECT INET6_ATON(NULL);
-- Returns: NULL
```

## Summary

`INET6_ATON()` and `INET6_NTOA()` are the preferred functions for handling IP addresses in MySQL applications that must support both IPv4 and IPv6. By converting addresses to compact binary values, you reduce storage footprint, keep indexes small, and enable efficient range-based subnet queries across both address families.
