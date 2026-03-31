# How to Use SET NAMES in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Character Set, Connection, SET NAMES, Encoding

Description: Learn how to use the SET NAMES statement in MySQL to configure the client connection character set and collation in a single command.

---

## What Is SET NAMES?

`SET NAMES` is a shorthand MySQL statement that sets three session variables simultaneously:

- `character_set_client` - Character set the client uses to send SQL statements.
- `character_set_connection` - Character set used for string literals within the connection.
- `character_set_results` - Character set MySQL uses when returning result sets to the client.

Running `SET NAMES 'utf8mb4'` is functionally equivalent to:

```sql
SET character_set_client     = utf8mb4;
SET character_set_connection = utf8mb4;
SET character_set_results    = utf8mb4;
```

## Basic Syntax

```sql
SET NAMES 'charset_name';
SET NAMES 'charset_name' COLLATE 'collation_name';
```

Examples:

```sql
-- Set to utf8mb4 with the server default collation
SET NAMES 'utf8mb4';

-- Set to utf8mb4 with an explicit collation
SET NAMES 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';

-- Restore server defaults
SET NAMES DEFAULT;
```

## When to Use SET NAMES

Call `SET NAMES` immediately after opening a connection when:
- The server default character set differs from what your application expects.
- You are using an older driver that does not support a `charset` connection option.
- You are running ad hoc scripts via the MySQL CLI and need to ensure correct encoding.

```bash
mysql -u root -p my_database --execute="SET NAMES utf8mb4; SELECT * FROM articles LIMIT 1;"
```

## Using SET NAMES in Application Code

Most modern drivers provide a native `charset` option that issues `SET NAMES` automatically. Use those when available.

For raw MySQL CLI scripts or legacy drivers:

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost', user='app', password='secret', database='shop'
)
cursor = conn.cursor()
cursor.execute("SET NAMES 'utf8mb4' COLLATE 'utf8mb4_unicode_ci'")
# Now all queries use utf8mb4
```

For PHP without PDO charset option:

```php
$pdo = new PDO('mysql:host=localhost;dbname=shop', 'app', 'secret');
$pdo->exec("SET NAMES 'utf8mb4' COLLATE 'utf8mb4_unicode_ci'");
```

## Difference Between SET NAMES and SET CHARACTER SET

`SET CHARACTER SET` is similar but also changes `character_set_database`:

```sql
-- SET CHARACTER SET additionally affects character_set_database
SET CHARACTER SET utf8mb4;
```

`SET NAMES` is the more commonly used statement and matches driver behavior. Use `SET NAMES` unless you specifically need `character_set_database` changed.

## Verifying the Effect

```sql
SET NAMES 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';

SHOW SESSION VARIABLES LIKE 'character_set%';
SHOW SESSION VARIABLES LIKE 'collation_connection';
```

Expected output:

```text
character_set_client     | utf8mb4
character_set_connection | utf8mb4
character_set_results    | utf8mb4
collation_connection     | utf8mb4_unicode_ci
```

## Summary

`SET NAMES` is the simplest way to configure the MySQL client-side encoding in a single statement. Always run it with `utf8mb4` as soon as a connection opens if the server default is not already `utf8mb4`. For production applications, prefer using the driver's built-in charset option so the setting is applied reliably on every connection.
