# How to Use ClickHouse with Prisma

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Prisma, TypeScript, ORM, Analytics

Description: Use Prisma's raw query API to interact with ClickHouse from TypeScript without fighting ORM limitations incompatible with columnar databases.

---

## Prisma and ClickHouse Compatibility

Prisma does not have a native ClickHouse connector. The recommended approach is to use Prisma for your OLTP database (PostgreSQL/MySQL) and use `$queryRawUnsafe` or a separate ClickHouse client for analytics queries. This hybrid gives you the best of both tools.

## Option 1 - Separate ClickHouse Client Alongside Prisma

```typescript
import { PrismaClient } from '@prisma/client';
import { createClient } from '@clickhouse/client';

const prisma = new PrismaClient();
const ch = createClient({
  url: 'http://localhost:8123',
  database: 'analytics',
});
```

Your transactional data lives in Postgres (via Prisma) and analytics events go to ClickHouse.

## Syncing Prisma Models to ClickHouse

After a Prisma write, fire an analytics event to ClickHouse.

```typescript
async function createOrder(data: OrderCreateInput) {
  const order = await prisma.order.create({ data });

  await ch.insert({
    table: 'order_events',
    values: [{
      order_id:   order.id,
      user_id:    order.userId,
      amount:     order.amount,
      created_at: order.createdAt.toISOString(),
    }],
    format: 'JSONEachRow',
  });

  return order;
}
```

## Querying ClickHouse for Aggregations

```typescript
async function getRevenueByDay(days: number) {
  const rs = await ch.query({
    query: `SELECT toDate(created_at)    AS day,
                   sum(amount)           AS revenue,
                   count()               AS orders
            FROM order_events
            WHERE created_at >= today() - {days:UInt32}
            GROUP BY day
            ORDER BY day`,
    query_params: { days },
    format: 'JSONEachRow',
  });
  return rs.json<{ day: string; revenue: string; orders: string }>();
}
```

## Option 2 - Prisma with a ClickHouse-Compatible MySQL Driver

Some teams use ClickHouse's MySQL-compatible port (9004) and point Prisma at it using the `mysql` provider. This approach is experimental and supports a subset of SQL.

```text
DATABASE_URL="mysql://default:@localhost:9004/analytics"
```

In `schema.prisma`:

```text
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}
```

Then use `$queryRaw` for ClickHouse-specific SQL.

## Combining Results in a Service

```typescript
async function dashboardData(userId: number) {
  const [profile, analytics] = await Promise.all([
    prisma.user.findUnique({ where: { id: userId } }),
    getRevenueByDay(30),
  ]);
  return { profile, analytics };
}
```

## Summary

Prisma does not natively support ClickHouse, but you can use both tools together - Prisma for OLTP operations and the `@clickhouse/client` for analytics queries. This separation keeps your transactional schema clean while giving full access to ClickHouse's analytical SQL.
