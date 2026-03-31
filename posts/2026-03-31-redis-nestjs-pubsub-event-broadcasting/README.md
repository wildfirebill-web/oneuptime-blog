# How to Use Redis Pub/Sub for NestJS Event Broadcasting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, NestJS, Pub/Sub, Event, Node.js

Description: Broadcast events across NestJS microservice instances using Redis Pub/Sub with a custom publisher service and subscriber listener.

---

When multiple NestJS instances need to react to the same event - like sending notifications to all connected users or invalidating a shared cache - Redis Pub/Sub broadcasts the message to every subscriber regardless of which instance publishes it.

## Install Dependencies

```bash
npm install ioredis
npm install @nestjs-modules/ioredis
```

## Configure Redis Module

```typescript
// app.module.ts
import { Module } from "@nestjs/common";
import { RedisModule } from "@nestjs-modules/ioredis";

@Module({
  imports: [
    RedisModule.forRoot({
      type: "single",
      url: process.env.REDIS_URL || "redis://localhost:6379",
    }),
  ],
})
export class AppModule {}
```

## Create a Publisher Service

```typescript
// events/event-publisher.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRedis } from "@nestjs-modules/ioredis";
import Redis from "ioredis";

@Injectable()
export class EventPublisherService {

  constructor(@InjectRedis() private readonly redis: Redis) {}

  async publish(channel: string, data: unknown): Promise<void> {
    await this.redis.publish(channel, JSON.stringify(data));
  }
}
```

## Create a Subscriber Service

```typescript
// events/event-subscriber.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from "@nestjs/common";
import Redis from "ioredis";

type Handler = (data: unknown) => void;

@Injectable()
export class EventSubscriberService implements OnModuleInit, OnModuleDestroy {

  private subscriber: Redis;
  private handlers = new Map<string, Handler[]>();

  onModuleInit() {
    this.subscriber = new Redis(process.env.REDIS_URL || "redis://localhost:6379");
    this.subscriber.on("message", (channel, message) => {
      const data = JSON.parse(message);
      (this.handlers.get(channel) || []).forEach((h) => h(data));
    });
  }

  async subscribe(channel: string, handler: Handler): Promise<void> {
    if (!this.handlers.has(channel)) {
      this.handlers.set(channel, []);
      await this.subscriber.subscribe(channel);
    }
    this.handlers.get(channel)!.push(handler);
  }

  async onModuleDestroy() {
    await this.subscriber.quit();
  }
}
```

## Use in a Feature Module

```typescript
// notifications/notifications.service.ts
import { Injectable, OnModuleInit } from "@nestjs/common";
import { EventPublisherService } from "../events/event-publisher.service";
import { EventSubscriberService } from "../events/event-subscriber.service";

@Injectable()
export class NotificationsService implements OnModuleInit {

  constructor(
    private publisher: EventPublisherService,
    private subscriber: EventSubscriberService
  ) {}

  async onModuleInit() {
    await this.subscriber.subscribe("user:notifications", (data) => {
      console.log("Notification received:", data);
    });
  }

  async broadcastNotification(userId: string, message: string) {
    await this.publisher.publish("user:notifications", { userId, message, ts: Date.now() });
  }
}
```

## Verify Pub/Sub Activity

```bash
redis-cli monitor | grep publish
redis-cli pubsub channels
```

## Summary

A dedicated `EventPublisherService` and `EventSubscriberService` wrap ioredis Pub/Sub in NestJS's dependency injection system, making it easy to broadcast events from any service. Each NestJS instance subscribes independently and receives a copy of every published message, enabling fan-out patterns for cache invalidation, real-time notifications, and cross-instance coordination without a dedicated message broker.
