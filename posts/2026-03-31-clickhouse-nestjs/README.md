# How to Use ClickHouse with NestJS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NestJS, TypeScript, Analytics, Database, Node.js

Description: Learn how to integrate ClickHouse into a NestJS application for high-performance analytics using the official ClickHouse Node.js client.

---

ClickHouse is an excellent choice for analytical workloads, and pairing it with NestJS gives you a strongly-typed, structured backend that can serve analytics at scale. This guide walks through connecting NestJS to ClickHouse and building a simple analytics service.

## Installing the ClickHouse Client

Install the official ClickHouse JavaScript client:

```bash
npm install @clickhouse/client
```

## Creating a ClickHouse Module

Create a dedicated module to manage the ClickHouse connection:

```typescript
// clickhouse.module.ts
import { Module, Global } from '@nestjs/common';
import { ClickHouseService } from './clickhouse.service';

@Global()
@Module({
  providers: [ClickHouseService],
  exports: [ClickHouseService],
})
export class ClickHouseModule {}
```

## Implementing the ClickHouse Service

```typescript
// clickhouse.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { createClient, ClickHouseClient } from '@clickhouse/client';

@Injectable()
export class ClickHouseService implements OnModuleInit {
  private client: ClickHouseClient;

  onModuleInit() {
    this.client = createClient({
      host: process.env.CLICKHOUSE_HOST || 'http://localhost:8123',
      username: process.env.CLICKHOUSE_USER || 'default',
      password: process.env.CLICKHOUSE_PASSWORD || '',
      database: process.env.CLICKHOUSE_DB || 'default',
    });
  }

  async query<T = unknown>(query: string): Promise<T[]> {
    const result = await this.client.query({
      query,
      format: 'JSONEachRow',
    });
    return result.json<T>();
  }

  async insert(table: string, values: Record<string, unknown>[]) {
    await this.client.insert({
      table,
      values,
      format: 'JSONEachRow',
    });
  }
}
```

## Creating an Analytics Table

```sql
CREATE TABLE IF NOT EXISTS events (
    event_id   UUID DEFAULT generateUUIDv4(),
    event_type String,
    user_id    UInt64,
    properties String,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY (created_at, user_id);
```

## Building an Analytics Controller

```typescript
// analytics.controller.ts
import { Controller, Get, Post, Body, Query } from '@nestjs/common';
import { ClickHouseService } from './clickhouse.service';

@Controller('analytics')
export class AnalyticsController {
  constructor(private readonly ch: ClickHouseService) {}

  @Post('event')
  async trackEvent(@Body() body: { type: string; userId: number }) {
    await this.ch.insert('events', [
      { event_type: body.type, user_id: body.userId, properties: '{}' },
    ]);
    return { status: 'ok' };
  }

  @Get('summary')
  async getSummary(@Query('from') from: string) {
    return this.ch.query(`
      SELECT event_type, count() AS total
      FROM events
      WHERE created_at >= toDateTime('${from}')
      GROUP BY event_type
      ORDER BY total DESC
      LIMIT 20
    `);
  }
}
```

## Registering the Module in AppModule

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { ClickHouseModule } from './clickhouse/clickhouse.module';
import { AnalyticsController } from './analytics/analytics.controller';

@Module({
  imports: [ClickHouseModule],
  controllers: [AnalyticsController],
})
export class AppModule {}
```

## Environment Configuration

Store credentials in your `.env` file and load them via `@nestjs/config`:

```text
CLICKHOUSE_HOST=http://localhost:8123
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=secret
CLICKHOUSE_DB=analytics
```

## Summary

Integrating ClickHouse into a NestJS application is straightforward with the official `@clickhouse/client` package. By building a global `ClickHouseService`, you can inject analytics capabilities into any controller or service. This pattern scales cleanly as your application grows.
