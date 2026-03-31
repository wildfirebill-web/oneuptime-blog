# How to Use ClickHouse with Bun Runtime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bun, TypeScript, Runtime, Analytics

Description: Connect to ClickHouse from Bun using the official Node.js-compatible client and take advantage of Bun's fast startup and built-in TypeScript support.

---

## Why Bun with ClickHouse

Bun is a fast JavaScript runtime with built-in TypeScript support and a Node.js-compatible module system. The `@clickhouse/client` package works in Bun without any transpilation step.

## Installation

```bash
bun add @clickhouse/client
```

## Creating a Client

```typescript
// src/db.ts
import { createClient } from '@clickhouse/client';

export const ch = createClient({
  url: process.env.CLICKHOUSE_URL ?? 'http://localhost:8123',
  database: 'analytics',
  username: 'default',
  password: '',
});
```

## Running Queries

```typescript
// src/index.ts
import { ch } from './db';

const rs = await ch.query({
  query: `SELECT event_name, count() AS cnt
          FROM events
          WHERE toDate(ts) = today()
          GROUP BY event_name
          ORDER BY cnt DESC
          LIMIT 10`,
  format: 'JSONEachRow',
});

const rows = await rs.json<{ event_name: string; cnt: string }>();
console.table(rows.map(r => ({ ...r, cnt: Number(r.cnt) })));

await ch.close();
```

Run it:

```bash
bun run src/index.ts
```

## Building a Bun HTTP Server

```typescript
import { ch } from './db';

Bun.serve({
  port: 3000,
  async fetch(req) {
    const url = new URL(req.url);
    if (url.pathname === '/api/events') {
      const rs = await ch.query({
        query: 'SELECT event_name, count() AS cnt FROM events GROUP BY event_name',
        format: 'JSONEachRow',
      });
      const data = await rs.json();
      return Response.json(data);
    }
    return new Response('Not Found', { status: 404 });
  },
});

console.log('Server running on http://localhost:3000');
```

## Streaming Inserts

```typescript
import { Readable } from 'stream';

const rows = Array.from({ length: 10_000 }, (_, i) => ({
  user_id: i,
  event_name: 'pageview',
  ts: new Date().toISOString(),
}));

await ch.insert({
  table: 'events',
  values: Readable.from(rows),
  format: 'JSONEachRow',
});
```

## Bun's File I/O for Bulk Loads

Bun's native file API is faster than Node.js's `fs`.

```typescript
const file = Bun.file('./events.ndjson');
const text = await file.text();
const rows = text.split('\n').filter(Boolean).map(JSON.parse);

await ch.insert({ table: 'events', values: rows, format: 'JSONEachRow' });
```

## Testing with Bun Test

```typescript
import { expect, test } from 'bun:test';

test('counts events', async () => {
  const rs = await ch.query({
    query: 'SELECT count() AS cnt FROM events',
    format: 'JSONEachRow',
  });
  const [row] = await rs.json<{ cnt: string }>();
  expect(Number(row.cnt)).toBeGreaterThanOrEqual(0);
});
```

## Summary

Bun's Node.js compatibility means the official `@clickhouse/client` works without modification. Bun's fast startup time makes it ideal for CLI analytics scripts and lightweight HTTP servers that query ClickHouse. Use `Bun.serve` for a minimal API server and `Bun.file` for fast local file loading.
