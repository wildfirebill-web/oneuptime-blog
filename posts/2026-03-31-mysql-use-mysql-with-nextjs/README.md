# How to Use MySQL with Next.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Next.js, Node.js

Description: Connect Next.js to MySQL using mysql2 and connection pooling, with examples for API routes, Server Actions, and environment variable management.

---

Next.js is a full-stack React framework that can query MySQL directly from API routes and Server Actions. Using `mysql2` with a singleton connection pool is the recommended pattern for serverless and long-running deployments alike.

## Installing the MySQL Driver

```bash
npm install mysql2
```

## Creating a Singleton Connection Pool

In serverless environments like Vercel, each function invocation can create a new connection unless you reuse a module-level pool. Create `lib/db.ts`:

```typescript
import mysql from 'mysql2/promise';

const pool = mysql.createPool({
  host:             process.env.MYSQL_HOST,
  port:             Number(process.env.MYSQL_PORT ?? 3306),
  user:             process.env.MYSQL_USER,
  password:         process.env.MYSQL_PASSWORD,
  database:         process.env.MYSQL_DATABASE,
  connectionLimit:  10,
  waitForConnections: true,
  queueLimit:       0,
});

export default pool;
```

## Environment Variables

```text
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=app_user
MYSQL_PASSWORD=strongpassword
MYSQL_DATABASE=app_db
```

Never commit `.env.local` to version control. Use Vercel environment variables or a secrets manager in production.

## API Route Example

```typescript
// app/api/users/route.ts
import { NextResponse } from 'next/server';
import pool from '@/lib/db';

export async function GET() {
  const [rows] = await pool.query('SELECT id, name, email FROM users LIMIT 20');
  return NextResponse.json(rows);
}
```

## Server Action Example

```typescript
// app/actions/createUser.ts
'use server';
import pool from '@/lib/db';

export async function createUser(name: string, email: string) {
  const [result] = await pool.execute(
    'INSERT INTO users (name, email) VALUES (?, ?)',
    [name, email]
  );
  return result;
}
```

Always use parameterized queries (`?` placeholders) to prevent SQL injection.

## Using an ORM (Prisma)

For larger projects, Prisma offers type-safe queries over MySQL:

```bash
npm install prisma @prisma/client
npx prisma init --datasource-provider mysql
```

```text
# prisma/schema.prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}
```

```text
DATABASE_URL="mysql://app_user:strongpassword@127.0.0.1:3306/app_db"
```

## Connection Limit in Serverless

Each Vercel function process shares one pool, but many concurrent function instances do not share pools. Limit total allowed connections by sizing `connectionLimit` conservatively (5-10) and setting `max_connections` on the MySQL server accordingly.

## Summary

Next.js connects to MySQL through `mysql2` pools or an ORM like Prisma. Keep credentials in environment variables, use parameterized queries everywhere, and set a conservative `connectionLimit` to avoid exhausting MySQL's `max_connections` in serverless deployments.
