# How to Handle ClickHouse Data Types in TypeScript

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, TypeScript, Data Type, Type Safety, JSON

Description: Map ClickHouse column types to TypeScript types correctly when using JSONEachRow format to avoid silent coercion bugs in analytics applications.

---

## The Serialization Challenge

When ClickHouse returns data as `JSONEachRow`, numeric types like `UInt64`, `Int64`, and `Decimal` are serialized as JSON strings to prevent precision loss. This means TypeScript `number` is wrong for large integers.

## Type Mapping Reference

| ClickHouse Type | JSON Value | TypeScript Type |
|---|---|---|
| UInt8, UInt16, UInt32 | number | `number` |
| UInt64 | string | `string` (or `BigInt`) |
| Int8..Int32 | number | `number` |
| Int64 | string | `string` (or `BigInt`) |
| Float32, Float64 | number | `number` |
| Decimal | string | `string` |
| String, FixedString | string | `string` |
| UUID | string | `string` |
| Date | string (YYYY-MM-DD) | `string` |
| DateTime | string | `string` |
| DateTime64 | string | `string` |
| Array(T) | array | `T[]` |
| Nullable(T) | value or null | `T \| null` |
| LowCardinality(String) | string | `string` |

## Defining Typed Interfaces

```typescript
interface PageViewRow {
  path:       string;
  cnt:        string;     // UInt64 - comes as string
  avg_ms:     number;     // Float64 - safe as number
  date:       string;     // Date - YYYY-MM-DD
  user_ids:   string[];   // Array(UInt64) - strings in array
}
```

## Parsing UInt64 Safely

```typescript
function parseUInt64(value: string): bigint {
  return BigInt(value);
}

const count = parseUInt64(row.cnt);
```

For display purposes you can convert to `number` if you know values fit.

```typescript
const displayCount = Number(row.cnt);
```

## Handling Nullable Fields

```typescript
interface EventRow {
  user_id:  number;
  region:   string | null;  // Nullable(String)
  score:    number | null;  // Nullable(Float64)
}

const region = row.region ?? 'unknown';
```

## DateTime Parsing

```typescript
const ts = new Date(row.created_at.replace(' ', 'T') + 'Z');
```

ClickHouse DateTime uses a space separator. Replace with `T` and append `Z` for UTC before passing to `Date`.

## Array Fields

```typescript
interface LogRow {
  tags:  string[];     // Array(String)
  codes: number[];     // Array(UInt32)
}

const hasError = row.codes.includes(500);
```

## Generic Query Helper with Types

```typescript
async function queryRows<T>(query: string, params = {}): Promise<T[]> {
  const rs = await ch.query({ query, query_params: params, format: 'JSONEachRow' });
  return rs.json<T>();
}

const rows = await queryRows<PageViewRow>(
  'SELECT path, count() AS cnt FROM page_views GROUP BY path'
);
```

## Summary

ClickHouse serializes UInt64 and Decimal as strings in JSON to preserve precision. Always type these fields as `string` in TypeScript and convert explicitly with `BigInt` or `Number`. Use `T | null` for Nullable columns and parse DateTime strings with a timezone suffix before constructing `Date` objects.
