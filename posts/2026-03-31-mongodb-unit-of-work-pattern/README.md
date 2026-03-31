# How to Implement the Unit of Work Pattern with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Unit Of Work, Transaction, Repository, Architecture

Description: Learn how to implement the Unit of Work pattern with MongoDB transactions to coordinate multiple repository operations in a single atomic unit.

---

The Unit of Work pattern tracks changes to multiple domain objects and coordinates writing them out as a single transaction. With MongoDB multi-document transactions, you can implement this pattern to ensure that a group of repository operations either all succeed or all fail atomically.

## The Problem Without Unit of Work

```javascript
// BAD - two separate operations, no atomicity
await orderRepo.create(order);          // succeeds
await inventoryRepo.decrement(item.id); // fails - inventory is now out of sync
```

## Unit of Work Interface

```typescript
// uow/IUnitOfWork.ts
export interface IUnitOfWork {
  begin(): Promise<void>;
  commit(): Promise<void>;
  rollback(): Promise<void>;
  getOrderRepository(): OrderRepository;
  getInventoryRepository(): InventoryRepository;
  getPaymentRepository(): PaymentRepository;
}
```

## MongoDB Unit of Work Implementation

```typescript
// uow/MongoUnitOfWork.ts
import mongoose, { ClientSession } from 'mongoose';
import { OrderRepository } from '../repositories/OrderRepository';
import { InventoryRepository } from '../repositories/InventoryRepository';
import { PaymentRepository } from '../repositories/PaymentRepository';

export class MongoUnitOfWork implements IUnitOfWork {
  private session: ClientSession | null = null;
  private orderRepo: OrderRepository | null = null;
  private inventoryRepo: InventoryRepository | null = null;
  private paymentRepo: PaymentRepository | null = null;

  async begin(): Promise<void> {
    this.session = await mongoose.startSession();
    this.session.startTransaction({
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority' },
    });
  }

  async commit(): Promise<void> {
    if (!this.session) throw new Error('Transaction not started');
    await this.session.commitTransaction();
    await this.session.endSession();
    this.session = null;
  }

  async rollback(): Promise<void> {
    if (!this.session) return;
    await this.session.abortTransaction();
    await this.session.endSession();
    this.session = null;
  }

  getOrderRepository(): OrderRepository {
    if (!this.session) throw new Error('Transaction not started');
    if (!this.orderRepo) {
      this.orderRepo = new OrderRepository(this.session);
    }
    return this.orderRepo;
  }

  getInventoryRepository(): InventoryRepository {
    if (!this.session) throw new Error('Transaction not started');
    if (!this.inventoryRepo) {
      this.inventoryRepo = new InventoryRepository(this.session);
    }
    return this.inventoryRepo;
  }

  getPaymentRepository(): PaymentRepository {
    if (!this.session) throw new Error('Transaction not started');
    if (!this.paymentRepo) {
      this.paymentRepo = new PaymentRepository(this.session);
    }
    return this.paymentRepo;
  }
}
```

## Session-Aware Repository Base

Extend the base repository to accept an optional session:

```typescript
// repositories/BaseRepository.ts
import { Model, Document, ClientSession } from 'mongoose';

export class BaseRepository<T extends Document> {
  constructor(
    protected model: Model<T>,
    protected session: ClientSession | null = null
  ) {}

  async create(data: Partial<T>): Promise<T> {
    const [doc] = await this.model.create([data], { session: this.session ?? undefined });
    return doc;
  }

  async updateById(id: string, update: object): Promise<T | null> {
    return this.model.findByIdAndUpdate(id, update, {
      new: true,
      session: this.session ?? undefined,
    }).lean() as Promise<T | null>;
  }
}
```

## Using Unit of Work in a Service

```typescript
// services/OrderService.ts
export class OrderService {
  constructor(private uowFactory: () => MongoUnitOfWork) {}

  async placeOrder(customerId: string, items: OrderItem[]) {
    const uow = this.uowFactory();
    await uow.begin();

    try {
      const orderRepo = uow.getOrderRepository();
      const inventoryRepo = uow.getInventoryRepository();
      const paymentRepo = uow.getPaymentRepository();

      // Verify and reserve inventory
      for (const item of items) {
        const reserved = await inventoryRepo.reserve(item.productId, item.quantity);
        if (!reserved) throw new Error(`Insufficient stock for product ${item.productId}`);
      }

      // Create order
      const total = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      const order = await orderRepo.create({ customerId, items, total, status: 'confirmed' });

      // Record payment intent
      await paymentRepo.create({ orderId: order._id, amount: total, status: 'pending' });

      await uow.commit();
      return order;
    } catch (err) {
      await uow.rollback();
      throw err;
    }
  }
}
```

## Wiring Up

```typescript
const orderService = new OrderService(() => new MongoUnitOfWork());

app.post('/api/orders', async (req, res) => {
  const order = await orderService.placeOrder(req.user.id, req.body.items);
  res.status(201).json(order);
});
```

## Summary

The Unit of Work pattern with MongoDB transactions ensures that inventory reservation, order creation, and payment recording either all commit or all roll back together. Pass the `ClientSession` into each repository so every operation participates in the same transaction. Encapsulate the session lifecycle (begin, commit, rollback) in the Unit of Work class to keep business logic clean and free of transaction management boilerplate.
