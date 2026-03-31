# How to Use ENCRYPT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Encrypt, Function, Security, Unix

Description: Learn about MySQL's ENCRYPT() function, its Unix crypt()-based implementation, deprecation status, and modern alternatives for password hashing.

---

## Introduction

The `ENCRYPT()` function in MySQL encrypts a string using the Unix `crypt()` system call. It accepts a string and an optional two-character salt and returns an encrypted string. This function was historically used for password storage in Unix-like systems, but it has been deprecated since MySQL 5.6.17 and removed in MySQL 8.0. Understanding its behavior is important when working with legacy MySQL 5.x databases.

## Basic Syntax

```sql
ENCRYPT(str [, salt])
```

- `str` - the string to encrypt
- `salt` - optional two-character string used to perturb the algorithm
- Returns a binary string, or `NULL` on systems where `crypt()` is not available

## Basic Examples (MySQL 5.7 and earlier)

```sql
-- Basic encryption with auto-generated salt
SELECT ENCRYPT('mypassword');
-- Returns something like: $1$abc12345$XbkuaFbBVm6o9Qzqs1YMZ/

-- With explicit two-char salt
SELECT ENCRYPT('mypassword', 'ab');
-- Returns: abBJNvVcS4nIQ
```

The output format depends on the operating system's `crypt()` implementation. On modern Linux with glibc, MD5 crypt or SHA-512 crypt may be used.

## Limitations

```sql
-- On Windows, ENCRYPT() always returns NULL
SELECT ENCRYPT('password');
-- Returns: NULL  (on Windows - crypt() not available)
```

Because `ENCRYPT()` depends on the OS-level `crypt()` function, its behavior varies across platforms. This makes it unsuitable for portable applications.

## Deprecation and Removal

In MySQL 5.6.17, `ENCRYPT()` was deprecated with a warning. In MySQL 8.0, it was removed entirely:

```sql
-- MySQL 8.0+
SELECT ENCRYPT('test');
-- ERROR 1305 (42000): FUNCTION mydb.ENCRYPT does not exist
```

## Checking if ENCRYPT() is Available

On MySQL 5.7:

```sql
SELECT @@version, ENCRYPT('test') IS NULL AS not_available;
```

## Modern Alternatives

### For Password Storage

Never store passwords with reversible encryption. Use one-way hashing with a secure algorithm at the application level:

```python
# Python - use bcrypt or Argon2
import bcrypt
hashed = bcrypt.hashpw(b"mypassword", bcrypt.gensalt(rounds=12))
```

### For Application-Level Hashing in MySQL

Use `SHA2()` for non-password checksums:

```sql
SELECT SHA2('data to hash', 256);
```

For AES encryption:

```sql
SET block_encryption_mode = 'aes-256-cbc';
SET @key = SHA2('passphrase', 256);
SET @iv = RANDOM_BYTES(16);
SELECT AES_ENCRYPT('sensitive value', @key, @iv);
```

### For Password Authentication Plugin

MySQL's built-in authentication uses `caching_sha2_password` (default in MySQL 8.0):

```sql
-- Create user with modern auth plugin
CREATE USER 'newuser'@'%' IDENTIFIED WITH caching_sha2_password BY 'Password1!';

-- Migrate old users
ALTER USER 'olduser'@'%' IDENTIFIED WITH caching_sha2_password BY 'NewPassword1!';
```

## Legacy Code Migration

If you encounter `ENCRYPT()` in MySQL 5.7 code being migrated to 8.0:

```sql
-- Old code (MySQL 5.7)
INSERT INTO users (username, pass_hash)
VALUES ('alice', ENCRYPT('password123', 'ab'));

-- New approach (MySQL 8.0) - hash at application level, store hash
-- Use bcrypt/Argon2 in application, store the resulting hash string
INSERT INTO users (username, pass_hash)
VALUES ('alice', '$2b$12$applicationGeneratedBcryptHash');
```

## Summary

MySQL's `ENCRYPT()` function was a thin wrapper around the Unix `crypt()` system call, useful only on Unix-like systems and deprecated as of MySQL 5.6.17, removed in MySQL 8.0. Legacy databases using `ENCRYPT()` should migrate password storage to application-level bcrypt or Argon2. For data encryption use cases, use `AES_ENCRYPT()` with CBC or GCM mode. For checksums, use `SHA2()`.
