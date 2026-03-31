# How to Use ClickHouse with Drizzle ORM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Drizzle ORM, TypeScript, SQL, Analytics

Description: Integrate ClickHouse with Drizzle ORM in TypeScript by leveraging the MySQL-compatible interface for schema definitions and raw SQL for analytics.

---

## Drizzle and ClickHouse

Drizzle ORM does not have a native ClickHouse driver, but ClickHouse exposes a MySQL-compatible port (9004). You can configure Drizzle's `mysql2` driver against it for basic table interactions, while using `sql` template literals and the official ClickHouse client for complex analytics queries.

## Installation

```bash
npm install drizzle-orm mysql2 @clickhouse/client
npm install -D drizzle-kit
```

## Drizzle Config via MySQL Port

```typescript
import { drizzle } from 'drizzle-orm/mysql2';
import mysql from 'mysql2/promise';

const pool = mysql.createPool({
  host: 'localhost',
  port: 9004,
  user: 'default',
  password: '',
  database: 'analytics',
});

export const db = drizzle(pool);
```

## Schema Definition

```typescript
import { mysqlTable, varchar, bigint, timestamp } from 'drizzle-orm/mysql-core';

export const events = mysqlTable('events', {
  userId:    bigint('user_id', { mode: 'number' }),
  eventName: varchar('event_name', { length: 255 }),
  ts:        timestamp('ts'),
});
```

## Basic Query with Drizzle

```typescript
import { desc, gte, sql } from 'drizzle-orm';

const cutoff = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
const results = await db
  .select({
    eventName: events.eventName,
    cnt: sql<number>`count()`,
  })
  .from(events)
  .where(gte(events.ts, cutoff))
  .groupBy(events.eventName)
  .orderBy(desc(sql`count()`))
  .limit(10);
```

## Mixing with the ClickHouse Native Client

For window functions, `arrayJoin`, or `toStartOfHour`, drop to the native client.

```typescript
import { createClient } from '@clickhouse/client';

const ch = createClient({ url: 'http://localhost:8123', database: 'analytics' });

async function hourlyBreakdown(days: number) {
  const rs = await ch.query({
    query: `SELECT toStartOfHour(ts) AS hour, count() AS cnt
            FROM events
            WHERE ts >= now() - INTERVAL {days:UInt32} DAY
            GROUP BY hour ORDER BY hour`,
    query_params: { days },
    format: 'JSONEachRow',
  });
  return rs.json<{ hour: string; cnt: string }>();
}
```

## Inserting Data

For ClickHouse inserts, the native client is faster.

```typescript
await ch.insert({
  table: 'events',
  values: rows,
  format: 'JSONEachRow',
});
```

## Closing Connections

```typescript
await pool.end();
await ch.close();
```

## Summary

Use Drizzle ORM's MySQL driver against ClickHouse's MySQL-compatible port for schema-driven queries and the official `@clickhouse/client` for analytical SQL. This hybrid approach gives you Drizzle's type-safe query builder for simple selects and full ClickHouse SQL power for aggregations.
