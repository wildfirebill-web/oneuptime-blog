# How to Use ClickHouse with NestJS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NestJS, TypeScript, Node.js, API, Analytics

Description: Integrate ClickHouse into a NestJS application using the clickhouse-client library to build fast analytics APIs with typed query results.

---

NestJS is a TypeScript-first Node.js framework for building scalable backend applications. Integrating ClickHouse lets you serve analytical query results through REST or GraphQL APIs with full type safety.

## Installing Dependencies

```bash
npm install @clickhouse/client
npm install --save-dev @types/node
```

## Creating a ClickHouse Module

Create `src/clickhouse/clickhouse.module.ts`:

```typescript
import { Module, Global } from '@nestjs/common';
import { ClickHouseService } from './clickhouse.service';

@Global()
@Module({
  providers: [ClickHouseService],
  exports: [ClickHouseService],
})
export class ClickHouseModule {}
```

## Creating the ClickHouse Service

Create `src/clickhouse/clickhouse.service.ts`:

```typescript
import { Injectable, OnModuleDestroy } from '@nestjs/common';
import { createClient, ClickHouseClient } from '@clickhouse/client';

@Injectable()
export class ClickHouseService implements OnModuleDestroy {
  private client: ClickHouseClient;

  constructor() {
    this.client = createClient({
      host: process.env.CLICKHOUSE_HOST || 'http://localhost:8123',
      username: process.env.CLICKHOUSE_USER || 'default',
      password: process.env.CLICKHOUSE_PASSWORD || '',
      database: process.env.CLICKHOUSE_DB || 'default',
    });
  }

  async query<T>(sql: string, params?: Record<string, unknown>): Promise<T[]> {
    const result = await this.client.query({
      query: sql,
      query_params: params,
      format: 'JSONEachRow',
    });
    return result.json<T>();
  }

  async insert<T extends Record<string, unknown>>(
    table: string,
    rows: T[]
  ): Promise<void> {
    await this.client.insert({
      table,
      values: rows,
      format: 'JSONEachRow',
    });
  }

  async onModuleDestroy() {
    await this.client.close();
  }
}
```

## Creating an Analytics Controller

Create `src/analytics/analytics.controller.ts`:

```typescript
import { Controller, Get, Query } from '@nestjs/common';
import { ClickHouseService } from '../clickhouse/clickhouse.service';

interface PageViewRow {
  page: string;
  views: string;
  unique_users: string;
}

@Controller('analytics')
export class AnalyticsController {
  constructor(private readonly clickhouse: ClickHouseService) {}

  @Get('top-pages')
  async getTopPages(@Query('days') days = '7') {
    const rows = await this.clickhouse.query<PageViewRow>(`
      SELECT
        page,
        count() AS views,
        uniq(user_id) AS unique_users
      FROM default.page_views
      WHERE created_at >= now() - INTERVAL {days:UInt8} DAY
      GROUP BY page
      ORDER BY views DESC
      LIMIT 20
    `, { days: Number(days) });

    return rows.map(r => ({
      page: r.page,
      views: Number(r.views),
      uniqueUsers: Number(r.unique_users),
    }));
  }
}
```

## Registering the Module

In `app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { ClickHouseModule } from './clickhouse/clickhouse.module';
import { AnalyticsController } from './analytics/analytics.controller';

@Module({
  imports: [ClickHouseModule],
  controllers: [AnalyticsController],
})
export class AppModule {}
```

## Summary

NestJS's dependency injection and module system make it easy to encapsulate ClickHouse as a shared service. The `@clickhouse/client` library provides a typed, promise-based API for running queries and inserting data. This pattern gives you a clean analytics API layer on top of ClickHouse's fast columnar storage.
