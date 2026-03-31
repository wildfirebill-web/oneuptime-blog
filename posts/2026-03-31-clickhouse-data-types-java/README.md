# How to Handle ClickHouse Data Types in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Java, Data Type, JDBC, Type Mapping

Description: Map ClickHouse data types to Java types correctly when using the JDBC driver to avoid runtime errors and silent data truncation.

---

## Type Mapping Reference

| ClickHouse Type | Java Type | JDBC Getter |
|---|---|---|
| UInt8, UInt16 | short / int | `getShort`, `getInt` |
| UInt32 | long | `getLong` |
| UInt64 | BigInteger or long | `getBigDecimal` |
| Int8..Int64 | byte..long | `getByte`..`getLong` |
| Float32 | float | `getFloat` |
| Float64 | double | `getDouble` |
| Decimal(p,s) | BigDecimal | `getBigDecimal` |
| String, FixedString | String | `getString` |
| UUID | String or UUID | `getString` then parse |
| Date | LocalDate | `getObject(n, LocalDate.class)` |
| DateTime | LocalDateTime | `getObject(n, LocalDateTime.class)` |
| DateTime64 | Instant or OffsetDateTime | `getObject` |
| Array(T) | Array | `getArray` |
| Nullable(T) | null-safe wrapper | check `wasNull()` |

## Reading UInt64 Safely

```java
BigDecimal raw = rs.getBigDecimal("event_count");
long count = raw.longValueExact();
```

Direct `getLong` on a UInt64 column may silently overflow for values above `Long.MAX_VALUE`.

## Reading DateTime64 with Timezone

```java
OffsetDateTime ts = rs.getObject("created_at", OffsetDateTime.class);
Instant instant = ts.toInstant();
```

## Handling Nullable Columns

```java
String region = rs.getString("region");
if (rs.wasNull()) {
    region = "unknown";
}
```

## Writing Java Types to ClickHouse

```java
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO metrics (id, value, recorded_at) VALUES (?, ?, ?)");

ps.setObject(1, UUID.randomUUID().toString());   // UUID as String
ps.setDouble(2, 3.14159);                        // Float64
ps.setObject(3, OffsetDateTime.now());           // DateTime with tz
ps.executeUpdate();
```

## Array Columns

Reading an `Array(String)` column:

```java
java.sql.Array arr = rs.getArray("tags");
String[] tags = (String[]) arr.getArray();
```

Writing an array:

```java
Array tags = conn.createArrayOf("String", new String[]{"infra", "alert"});
ps.setArray(1, tags);
```

## LowCardinality and Enum Types

`LowCardinality(String)` maps to `String` in Java. `Enum8` and `Enum16` map to `String` (the label) by default.

```java
String status = rs.getString("status");  // returns label, e.g., "active"
```

## Summary

Correct type mapping prevents runtime exceptions and data loss when reading from or writing to ClickHouse via JDBC. Pay special attention to UInt64 (use BigDecimal), DateTime64 (use OffsetDateTime), Nullable columns (check wasNull), and Array columns (cast the SQL Array object).
