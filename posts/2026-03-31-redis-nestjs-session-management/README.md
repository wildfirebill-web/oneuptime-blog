# How to Use Redis for NestJS Session Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, NestJS, Session, Authentication, Node.js

Description: Store NestJS sessions in Redis using express-session and connect-redis so sessions persist across restarts and are shared across all instances.

---

NestJS is built on top of Express, which means `express-session` and `connect-redis` work natively. This setup stores session data in Redis instead of memory, enabling stateless horizontal scaling.

## Install Dependencies

```bash
npm install express-session connect-redis redis
npm install -D @types/express-session
```

## Configure Session Middleware in main.ts

```typescript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import * as session from "express-session";
import RedisStore from "connect-redis";
import { createClient } from "redis";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const redisClient = createClient({ url: process.env.REDIS_URL || "redis://localhost:6379" });
  redisClient.on("error", (err) => console.error("Redis error", err));
  await redisClient.connect();

  app.use(
    session({
      store: new RedisStore({ client: redisClient }),
      secret: process.env.SESSION_SECRET!,
      resave: false,
      saveUninitialized: false,
      cookie: {
        secure: process.env.NODE_ENV === "production",
        httpOnly: true,
        maxAge: 30 * 60 * 1000, // 30 minutes
      },
    })
  );

  await app.listen(3000);
}
bootstrap();
```

## Access Session in Controllers

```typescript
import { Controller, Post, Get, Body, Req, Res } from "@nestjs/common";
import { Request, Response } from "express";

@Controller("auth")
export class AuthController {

  @Post("login")
  async login(@Body() body: LoginDto, @Req() req: Request) {
    const user = await this.authService.validate(body.username, body.password);
    if (!user) return { error: "Invalid credentials" };

    req.session["userId"] = user.id;
    req.session["username"] = user.username;
    return { message: "Logged in" };
  }

  @Get("me")
  getProfile(@Req() req: Request) {
    if (!req.session["userId"]) {
      return { error: "Not authenticated" };
    }
    return { userId: req.session["userId"], username: req.session["username"] };
  }

  @Post("logout")
  logout(@Req() req: Request, @Res() res: Response) {
    req.session.destroy((err) => {
      res.clearCookie("connect.sid");
      res.json({ message: "Logged out" });
    });
  }
}
```

## Create a Session Guard

```typescript
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from "@nestjs/common";

@Injectable()
export class SessionGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    if (!request.session?.userId) {
      throw new UnauthorizedException("Session required");
    }
    return true;
  }
}
```

Apply it to protected routes:

```typescript
@Get("profile")
@UseGuards(SessionGuard)
getProfile(@Req() req: Request) {
  return { userId: req.session["userId"] };
}
```

## Inspect Sessions in Redis

```bash
redis-cli keys "sess:*"
redis-cli get "sess:abc123xyz"
redis-cli ttl "sess:abc123xyz"
```

## Summary

NestJS inherits Express session middleware, so `connect-redis` integrates with zero framework-specific code. Configuring `RedisStore` in `main.ts` redirects all session reads and writes to Redis, enabling persistent sessions and shared state across multiple NestJS instances. A custom `SessionGuard` protects routes cleanly using NestJS's standard guard system.
