---
name: backend-nestjs-vertical-hexagonal
description: Enforces a Vertical Slice / Feature-First architecture with an internal Hexagonal Architecture (Ports and Adapters) structure and Domain-Driven Design (DDD) for NestJS backend projects.
---

# Enterprise NestJS Vertical-Hexagonal & DDD Guideline

This guideline defines the industry-standard implementation of **Hexagonal Architecture**, **Vertical Slicing (Feature-First)**, and **Domain-Driven Design (DDD)** in enterprise NestJS projects, matching patterns used by tech leaders.

---

## 1. Core Architectural Pillars

1.  **Screaming Architecture**: Folder structures scream the domain features (e.g., `orders`, `inventory`, `billing`) instead of technical frameworks.
2.  **Pure Domain (Core)**: The inner circle (`domain/` and `application/`) is written in pure TypeScript. It has **zero dependencies** on NestJS (`@nestjs/common`), database tools (Prisma, TypeORM, mongoose), or any external framework.
3.  **Strict Dependency Direction**: Outer layers import from inner layers. Inner layers only import from themselves or the `shared/` kernel.
    *   `Presentation -> Application -> Domain`
    *   `Infrastructure -> Application -> Domain`
4.  **No Anemic Domains**: Entities must protect their business rules (invariants) through encapsulated properties, self-validations, and Value Objects.
5.  **Persistence Decoupling (Mappers)**: Database entities are mapped to/from pure domain models. ORM schemas (Prisma types, TypeORM decorators) never enter the Domain layer.

---

## 2. Directory Layouts (Monolith vs. Microservice)

### A. Monolithic Structure
Features reside under `src/modules/`. Cross-cutting domain concerns (like a base `Entity` class or shared `Email` value object) reside in `src/shared/`.
```text
src/
├── main.ts
├── app.module.ts
├── shared/                             # Shared Kernel
│   ├── domain/
│   │   ├── building-blocks/            # DDD Base Primitives
│   │   │   ├── Entity.ts
│   │   │   ├── AggregateRoot.ts
│   │   │   └── ValueObject.ts
│   │   └── value-objects/
│   │       └── Email.ts
│   └── infrastructure/
│       └── prisma/
│           └── prisma.service.ts
└── modules/
    └── orders/                         # Bounded Context / Vertical Slice
        ├── orders.module.ts
        ├── domain/
        │   ├── entities/
        │   │   └── Order.ts
        │   ├── value-objects/
        │   │   └── OrderId.ts
        │   ├── exceptions/
        │   │   └── OrderErrors.ts
        │   └── ports/
        │       └── OrderRepository.ts  # Repository interface (Port)
        ├── application/
        │   ├── dtos/
        │   │   └── CreateOrderInput.ts
        │   └── use-cases/
        │       └── CreateOrderUseCase.ts
        ├── infrastructure/
        │   ├── persistence/
        │   │   ├── PrismaOrderRepository.ts # Adapter implementation
        │   │   └── OrderMapper.ts       # Domain <-> DB entity translator
        │   └── config/
        └── presentation/
            ├── controllers/
            │   └── OrderController.ts   # HTTP or gRPC Controller
            └── dtos/
                └── CreateOrderDto.ts   # class-validator schemas
```

### B. Microservice Structure
In a microservice, the entire repository is dedicated to a single Bounded Context. Features are sliced vertically, but there is no `modules/` nesting layer:
```text
src/
├── main.ts
├── app.module.ts
├── shared/                             # Shared Kernel (base DDD and utils)
└── auth/                               # Vertical module directory
    ├── auth.module.ts
    ├── domain/
    ├── application/
    ├── infrastructure/
    └── presentation/
```

---

## 3. DDD Base Primitives (Enterprise Style)

Define base classes in `src/shared/domain/building-blocks/` to avoid boilerplate and enforce consistency.

### A. Value Object Base Class
`src/shared/domain/building-blocks/ValueObject.ts`
```typescript
export abstract class ValueObject<T> {
  protected readonly props: T;

  constructor(props: T) {
    this.props = Object.freeze(props);
  }

  public equals(vo?: ValueObject<T>): boolean {
    if (vo === null || vo === undefined) return false;
    return JSON.stringify(this.props) === JSON.stringify(vo.props);
  }
}
```

### B. Entity Base Class
`src/shared/domain/building-blocks/Entity.ts`
```typescript
export abstract class Entity<T> {
  protected readonly _id: string;
  protected readonly props: T;

  constructor(props: T, id?: string) {
    this._id = id ?? crypto.randomUUID();
    this.props = props;
  }

  get id(): string {
    return this._id;
  }

  public equals(entity?: Entity<T>): boolean {
    if (entity === null || entity === undefined) return false;
    if (this === entity) return true;
    return this._id === entity.id;
  }
}
```

### C. Aggregate Root & Domain Events
`src/shared/domain/building-blocks/AggregateRoot.ts`
```typescript
import { Entity } from './Entity';

export abstract class AggregateRoot<T> extends Entity<T> {
  private _domainEvents: any[] = [];

  get domainEvents(): any[] {
    return this._domainEvents;
  }

  protected addDomainEvent(event: any): void {
    this._domainEvents.push(event);
  }

  public clearDomainEvents(): void {
    this._domainEvents = [];
  }
}
```

---

## 4. Decoupling Base Model and Persistence (Mappers)

Do not leak ORM annotations or database schemas into your domain. The Domain Entity represents pure business rules, while the DB Adapter uses a **Mapper** to translate.

### A. Domain Entity (Rich & Clean)
`src/modules/orders/domain/entities/Order.ts`
```typescript
import { AggregateRoot } from '../../../../shared/domain/building-blocks/AggregateRoot';
import { ValueObject } from '../../../../shared/domain/building-blocks/ValueObject';

interface OrderProps {
  customerId: string;
  price: number;
  status: string;
}

export class Order extends AggregateRoot<OrderProps> {
  constructor(props: OrderProps, id?: string) {
    // Enforce business rules
    if (props.price < 0) throw new Error('Price cannot be negative');
    super(props, id);
  }

  get customerId(): string { return this.props.customerId; }
  get price(): number { return this.props.price; }
  get status(): string { return this.props.status; }

  public completeOrder(): void {
    this.props.status = 'COMPLETED';
    this.addDomainEvent({ event: 'order.completed', orderId: this.id });
  }
}
```

### B. Infrastructure Mapper Example
`src/modules/orders/infrastructure/persistence/OrderMapper.ts`
```typescript
import { Order as PrismaOrder } from '@prisma/client';
import { Order } from '../../domain/entities/Order';

export class OrderMapper {
  static toDomain(raw: PrismaOrder): Order {
    return new Order(
      {
        customerId: raw.customerId,
        price: raw.price,
        status: raw.status,
      },
      raw.id,
    );
  }

  static toPersistence(domain: Order) {
    return {
      id: domain.id,
      customerId: domain.customerId,
      price: domain.price,
      status: domain.status,
    };
  }
}
```

---

## 5. Dependency Injection without Framework Leakage

Use cases must not contain `@Injectable()` or `@Inject()`. They receive ports in their constructors and are wired in the NestJS Module.

### A. Core Use Case
`src/modules/orders/application/use-cases/CreateOrderUseCase.ts`
```typescript
import { Order } from '../../domain/entities/Order';
import { OrderRepository } from '../../domain/ports/OrderRepository';

export class CreateOrderUseCase {
  constructor(private readonly orderRepository: OrderRepository) {}

  async execute(customerId: string, price: number): Promise<string> {
    const order = new Order({ customerId, price, status: 'PENDING' });
    await this.orderRepository.save(order);
    return order.id;
  }
}
```

### B. Module Composition Root
`src/modules/orders/orders.module.ts`
```typescript
import { Module } from '@nestjs/common';
import { CreateOrderUseCase } from './application/use-cases/CreateOrderUseCase';
import { OrderController } from './presentation/controllers/OrderController';
import { PrismaOrderRepository } from './infrastructure/persistence/PrismaOrderRepository';

export const ORDER_REPOSITORY_TOKEN = Symbol('ORDER_REPOSITORY');

@Module({
  controllers: [OrderController],
  providers: [
    {
      provide: ORDER_REPOSITORY_TOKEN,
      useClass: PrismaOrderRepository,
    },
    {
      provide: CreateOrderUseCase,
      useFactory: (repo) => new CreateOrderUseCase(repo),
      inject: [ORDER_REPOSITORY_TOKEN],
    },
  ],
})
export class OrdersModule {}
```
