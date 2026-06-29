---
name: backend-nestjs-vertical-hexagonal
description: Enforces a Vertical Slice / Feature-First architecture with an internal Hexagonal Architecture (Ports and Adapters) structure and Domain-Driven Design (DDD) for NestJS backend projects.
---

# Enterprise NestJS Vertical-Hexagonal & DDD Guideline

This skill defines the implementation of **Hexagonal Architecture**, **Vertical Slicing (Feature-First)**, and **Domain-Driven Design (DDD)** in enterprise NestJS projects, validated via `eslint-plugin-hexagonal-architecture`.

---

## 1. Core Architectural Pillars & ESLint Boundaries

1.  **Screaming Architecture**: Folder structures scream the domain features (e.g., `orders`, `inventory`, `billing`) instead of technical frameworks.
2.  **Pure Domain (Core)**: The inner circle (`domain/` and `application/`) is written in pure TypeScript. It has **zero dependencies** on NestJS (`@nestjs/common`), database tools (Prisma, TypeORM, mongoose), or any external framework.
3.  **Strict Dependency Direction**: Outer layers import from inner layers. Inner layers only import from themselves or the `shared/` kernel.
4.  **Mandatory ESLint Rule Boundaries**: The project must install and configure `eslint-plugin-hexagonal-architecture`.
    *   **Domain (`domain/`)**: Can only import from within the same `domain/` folder or shared domain utilities. It has no external dependencies.
    *   **Application (`application/`)**: Can only import from `domain/` and other `application/` cases. It cannot import from `infrastructure/` or presentation.
    *   **Infrastructure (`infrastructure/`)**: Can import from `domain/`, `application/`, and other `infrastructure/` files.
    *   **Presentation (`presentation/`)**: The controller/routing layer resides outside the inner circle, allowing controllers to import from `application/` and `domain/` without triggering ESLint errors.

### ESLint Configuration (`.eslintrc.js` or equivalent)
```javascript
module.exports = {
  plugins: ["hexagonal-architecture"],
  overrides: [
    {
      files: ["src/modules/**/*.ts"],
      rules: {
        "hexagonal-architecture/enforce": ["error"]
      }
    }
  ]
};
```

---

## 2. Directory Layout (Monolith Spec)

Features reside under `src/modules/`. Cross-cutting domain concerns (like base DDD classes, shared exceptions or shared value objects) reside in `src/shared/`.

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
│   │   ├── exceptions/
│   │   │   └── DomainError.ts          # Base Domain Error class
│   │   └── value-objects/
│   │       └── Email.ts                # Reusable Value Object
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
        │   │   └── OrderErrors.ts      # Feature-specific Domain Errors
        │   └── ports/
        │       ├── OrderRepository.ts  # Repository interface (Port)
        │       └── tokens.ts           # DI Tokens (Symbol)
        ├── application/
        │   ├── dtos/
        │   │   └── CreateOrderInput.ts
        │   └── use-cases/
        │       └── CreateOrderUseCase.ts
        ├── infrastructure/
        │   ├── adapters/
        │   │   ├── repository/
        │   │   │   └── PrismaOrderRepository.ts # Adapter implementation
        │   │   └── mappers/
        │   │       └── OrderMapper.ts       # Domain <-> DB mapper
        │   └── config/
        └── presentation/
            ├── controllers/
            │   └── OrderController.ts   # HTTP Controller
            ├── filters/
            │   └── OrderExceptionFilter.ts # Maps domain errors to HTTP
            └── dtos/
                └── CreateOrderDto.ts   # class-validator schemas
```

---

## 3. DDD Base Primitives & Shared Errors

Define base classes in `src/shared/domain/building-blocks/` and base exceptions in `src/shared/domain/exceptions/`.

### A. Value Object Base Class
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
```typescript
import { Entity } from './Entity';

export abstract class AggregateRoot<T> extends Entity<T> {
  private _domainEvents: unknown[] = [];

  get domainEvents(): unknown[] {
    return this._domainEvents;
  }

  protected addDomainEvent(event: unknown): void {
    this._domainEvents.push(event);
  }

  public clearDomainEvents(): void {
    this._domainEvents = [];
  }
}
```

### D. Base Domain Exception
`src/shared/domain/exceptions/DomainError.ts`
```typescript
export abstract class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}
```

---

## 4. Rich Domain & Persistence Decoupling (Mappers)

ORM decorators (Prisma schemas, TypeORM annotations) are infrastructure details. They must not enter the `domain` or `application` layers. Use a **Mapper** to translate.

### A. Feature Domain Errors
`src/modules/orders/domain/exceptions/OrderErrors.ts`
```typescript
import { DomainError } from '../../../../shared/domain/exceptions/DomainError';

export class OrderNotFoundError extends DomainError {
  constructor(orderId: string) {
    super(`Order with ID ${orderId} was not found`);
  }
}

export class InvalidOrderPriceError extends DomainError {
  constructor(price: number) {
    super(`Price ${price} is invalid, must be positive`);
  }
}
```

### B. Domain Entity (Rich & Clean)
`src/modules/orders/domain/entities/Order.ts`
```typescript
import { AggregateRoot } from '../../../../shared/domain/building-blocks/AggregateRoot';
import { InvalidOrderPriceError } from '../exceptions/OrderErrors';

interface OrderProps {
  customerId: string;
  price: number;
  status: string;
}

export class Order extends AggregateRoot<OrderProps> {
  constructor(props: OrderProps, id?: string) {
    if (props.price < 0) {
      throw new InvalidOrderPriceError(props.price);
    }
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

### C. Infrastructure Mapper
`src/modules/orders/infrastructure/adapters/mappers/OrderMapper.ts`
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

  static toPersistence(domain: Order): Omit<PrismaOrder, 'createdAt' | 'updatedAt'> {
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

Use cases must be pure TypeScript classes without `@Injectable()` or `@Inject()` decorators. They receive repositories (ports) via constructor injection. Wiring is handled at the NestJS Module level using token-based injection.

### A. Dedicated DI Tokens
`src/modules/orders/domain/ports/tokens.ts`
```typescript
export const ORDER_REPOSITORY = Symbol('ORDER_REPOSITORY');
```

### B. Use Case
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

### C. NestJS Module configuration
`src/modules/orders/orders.module.ts`
```typescript
import { Module } from '@nestjs/common';
import { CreateOrderUseCase } from './application/use-cases/CreateOrderUseCase';
import { OrderController } from './presentation/controllers/OrderController';
import { PrismaOrderRepository } from './infrastructure/adapters/repository/PrismaOrderRepository';
import { ORDER_REPOSITORY } from './domain/ports/tokens';

@Module({
  controllers: [OrderController],
  providers: [
    {
      provide: ORDER_REPOSITORY,
      useClass: PrismaOrderRepository,
    },
    {
      provide: CreateOrderUseCase,
      useFactory: (repo) => new CreateOrderUseCase(repo),
      inject: [ORDER_REPOSITORY],
    },
  ],
})
export class OrdersModule {}
```

---

## 6. Presentation Exception Mapping

To prevent leaking raw database errors or framework-specific messages, catch domain errors and map them to appropriate HTTP/gRPC statuses using NestJS Exception Filters in `presentation/filters/`.

Avoid checking exception message strings (`exception.message.includes('not found')`) as it is fragile and breaks silently when messages change. Instead, catch `DomainError` classes specifically and map error constructors using a TypeScript `Map`.

### A. Exception Filter
`src/modules/orders/presentation/filters/OrderExceptionFilter.ts`
```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpStatus } from '@nestjs/common';
import { Response } from 'express';
import { DomainError } from '../../../../shared/domain/exceptions/DomainError';
import { OrderNotFoundError, InvalidOrderPriceError } from '../../domain/exceptions/OrderErrors';

type DomainErrorConstructor = new (...args: any[]) => DomainError;

// Map domain error classes directly to HTTP Status Codes
const ERROR_STATUS_MAP = new Map<DomainErrorConstructor, HttpStatus>([
  [OrderNotFoundError, HttpStatus.NOT_FOUND],
  [InvalidOrderPriceError, HttpStatus.BAD_REQUEST],
]);

@Catch(DomainError) // Catch only domain errors, leaving systemic exceptions (like HTTP 500s) to global filters
export class OrderExceptionFilter implements ExceptionFilter {
  catch(exception: DomainError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    const errorClass = exception.constructor as DomainErrorConstructor;
    const status = ERROR_STATUS_MAP.get(errorClass) ?? HttpStatus.BAD_REQUEST;

    response.status(status).json({
      statusCode: status,
      error: exception.name,
      message: exception.message,
      timestamp: new Date().toISOString(),
    });
  }
}
```

### B. Controller Integration
`src/modules/orders/presentation/controllers/OrderController.ts`
```typescript
import { Controller, Post, Body, UseFilters } from '@nestjs/common';
import { CreateOrderUseCase } from '../../application/use-cases/CreateOrderUseCase';
import { CreateOrderDto } from '../dtos/CreateOrderDto';
import { OrderExceptionFilter } from '../filters/OrderExceptionFilter';

@Controller('orders')
@UseFilters(OrderExceptionFilter)
export class OrderController {
  constructor(private readonly createOrderUseCase: CreateOrderUseCase) {}

  @Post()
  async create(@Body() dto: CreateOrderDto) {
    const id = await this.createOrderUseCase.execute(dto.customerId, dto.price);
    return { id };
  }
}
```
