# How to Use BINARY Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Data Type, Binary, Storage

Description: Learn how the fixed-length BINARY data type works in MySQL, how it differs from CHAR, and practical use cases for binary storage.

---

## What Is the BINARY Data Type

`BINARY(n)` stores fixed-length binary strings of exactly `n` bytes. It is the binary counterpart of `CHAR(n)`. MySQL pads shorter values with `0x00` bytes on the right and always stores exactly `n` bytes.

The maximum length is 255 bytes.

## Declaring a BINARY Column

```sql
CREATE TABLE sessions (
    session_id BINARY(16) NOT NULL,
    user_id    INT UNSIGNED NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (session_id)
);
```

## Inserting Binary Data

```sql
-- Store a UUID in binary form (16 bytes, more compact than CHAR(36))
INSERT INTO sessions (session_id, user_id)
VALUES (UUID_TO_BIN(UUID()), 42);

-- Using a hex literal
INSERT INTO sessions (session_id, user_id)
VALUES (0x4a6f686e446f65000000000000000000, 7);
```

## Retrieving Binary Data

```sql
-- Display as hex
SELECT HEX(session_id), user_id FROM sessions;

-- Convert back to UUID string
SELECT BIN_TO_UUID(session_id) AS uuid, user_id FROM sessions;
```

## BINARY vs CHAR: Key Differences

| Feature | BINARY(n) | CHAR(n) |
|---|---|---|
| Comparison | Byte-by-byte | Collation-aware |
| Padding byte | 0x00 (null byte) | 0x20 (space) |
| Case-sensitive | Always | Depends on collation |
| Charset | Binary | Table/column charset |

Binary comparisons are case-sensitive and locale-independent, which is useful for hashes and cryptographic values.

## Case-Sensitive Comparisons

```sql
CREATE TABLE tokens (token BINARY(32));
INSERT INTO tokens VALUES ('ABC'), ('abc');

-- BINARY comparison distinguishes case
SELECT * FROM tokens WHERE token = 'ABC';
-- Returns only 'ABC', not 'abc'
```

## Storing Password Hashes

`BINARY` is appropriate for fixed-length hashes:

```sql
CREATE TABLE credentials (
    user_id      INT UNSIGNED PRIMARY KEY,
    password_hash BINARY(32)  -- SHA-256 = 32 bytes
);

INSERT INTO credentials (user_id, password_hash)
VALUES (1, UNHEX(SHA2('my_secret_password', 256)));

-- Verify
SELECT user_id
FROM credentials
WHERE password_hash = UNHEX(SHA2('my_secret_password', 256));
```

## Padding Behavior

Values shorter than `n` bytes are right-padded with `0x00`:

```sql
CREATE TABLE padded (val BINARY(5));
INSERT INTO padded VALUES ('ab');
SELECT HEX(val), LENGTH(val) FROM padded;
-- HEX: 6162000000, LENGTH: 5
```

## Summary

`BINARY(n)` is a fixed-length binary type best suited for cryptographic hashes, UUIDs in compact binary form, and other fixed-size binary identifiers. Its byte-by-byte, case-sensitive comparison makes it more predictable than character types for security-critical data. For variable-length binary data, use `VARBINARY`.
