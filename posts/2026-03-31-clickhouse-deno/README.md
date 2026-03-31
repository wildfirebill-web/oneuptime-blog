# How to Use ClickHouse with Deno

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Deno, TypeScript, HTTP Client, Analytics

Description: Query ClickHouse from Deno using the HTTP API directly or the official client configured for Node.js compatibility mode.

---

## Two Approaches

1. Use ClickHouse's HTTP API directly with `fetch` - no dependencies needed.
2. Use the official `@clickhouse/client-web` package via npm specifier.

## Approach 1 - Native fetch to ClickHouse HTTP API

ClickHouse exposes an HTTP interface on port 8123. You can query it with any HTTP client.

```typescript
const CLICKHOUSE = 'http://localhost:8123';

async function query<T>(sql: string): Promise<T[]> {
  const url = new URL(CLICKHOUSE);
  url.searchParams.set('database', 'analytics');
  url.searchParams.set('query', sql);
  url.searchParams.set('default_format', 'JSONEachRow');

  const res = await fetch(url, {
    headers: {
      'X-ClickHouse-User': 'default',
      'X-ClickHouse-Key': '',
    },
  });

  if (!res.ok) {
    throw new Error(`ClickHouse error: ${await res.text()}`);
  }

  const text = await res.text();
  return text.trim().split('\n').filter(Boolean).map(line => JSON.parse(line) as T);
}
```

## Running a Query

```typescript
interface TopEvent {
  event_name: string;
  cnt: string;
}

const results = await query<TopEvent>(
  `SELECT event_name, count() AS cnt
   FROM events
   WHERE toDate(ts) = today()
   GROUP BY event_name
   ORDER BY cnt DESC
   LIMIT 10`
);

for (const row of results) {
  console.log(`${row.event_name}: ${row.cnt}`);
}
```

## Approach 2 - Official Web Client via npm

```typescript
import { createClient } from 'npm:@clickhouse/client-web';

const ch = createClient({
  url: 'http://localhost:8123',
  database: 'analytics',
  username: 'default',
  password: '',
});

const rs = await ch.query({
  query: 'SELECT count() FROM events',
  format: 'JSONEachRow',
});

const [row] = await rs.json<{ 'count()': string }>();
console.log('Total events:', row['count()']);
await ch.close();
```

## POST Insert via fetch

```typescript
async function insert(table: string, rows: object[]) {
  const body = rows.map(r => JSON.stringify(r)).join('\n');
  const url = `${CLICKHOUSE}/?query=INSERT+INTO+${table}+FORMAT+JSONEachRow`;

  const res = await fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-ClickHouse-User': 'default',
      'X-ClickHouse-Key': '',
    },
    body,
  });

  if (!res.ok) throw new Error(await res.text());
}
```

## Deno Oak Server with ClickHouse

```typescript
import { Application, Router } from 'https://deno.land/x/oak/mod.ts';

const router = new Router();

router.get('/api/events', async (ctx) => {
  const rows = await query('SELECT event_name, count() AS cnt FROM events GROUP BY event_name');
  ctx.response.body = rows;
});

const app = new Application();
app.use(router.routes());
await app.listen({ port: 3000 });
```

## Summary

Deno can interact with ClickHouse via the HTTP API using the built-in `fetch`, requiring zero dependencies. For a richer API, use `npm:@clickhouse/client-web` with Deno's npm compatibility. Both approaches provide full access to ClickHouse queries and inserts from Deno.
