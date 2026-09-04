---
name: backend-nestjs-modular
description: Guidelines for building clean, modular NestJS APIs with Vertical Slicing and Screaming Architecture. Feature-first modules with Controller → Service → Repository pattern. Supports Prisma. Use when the user wants modular backend architecture without hexagonal/DDD overhead.
---

# NestJS Modular Architecture (Vertical Slice + Clean Code)

Pragmatic **Feature-First / Vertical Slice** architecture for NestJS APIs. No hexagonal layers, no ports & adapters, no DI tokens, no ESLint architecture plugin — just clean, reusable, module-based NestJS code.

---

## 1. Core Principles

1. **Screaming Architecture**: Folder names reflect business features (`orders`, `products`, `billing`), not technical layers.
2. **One Module = One Feature**: Each feature is self-contained with its own controller, service, DTOs, and optional repository.
3. **Controller → Service → Repository**: Simple, idiomatic NestJS flow. Business logic lives in the **service**. Data access lives in the **repository** (or directly in service for simple CRUD).
4. **DTOs at the Edge**: Use `class-validator` DTOs for request/response validation at the controller boundary.
5. **Shared Kernel**: Cross-cutting concerns (Prisma, guards, filters, pipes) live in `src/shared/`.

---

## 2. Directory Layout

```text
src/
├── main.ts
├── app.module.ts
├── shared/
│   ├── prisma/
│   │   └── prisma.service.ts
│   ├── filters/
│   │   └── http-exception.filter.ts
│   ├── guards/
│   │   └── auth.guard.ts
│   └── pipes/
│       └── validation.pipe.ts
└── modules/
    └── products/
        ├── products.module.ts
        ├── products.controller.ts
        ├── products.service.ts
        ├── products.repository.ts       # Optional: extract when queries grow
        └── dto/
            ├── create-product.dto.ts
            ├── update-product.dto.ts
            └── product-response.dto.ts
```

---

## 3. Layer Responsibilities

| Layer | Responsibility |
|-------|---------------|
| **Controller** | HTTP routing, request validation (DTOs), response mapping. No business logic. |
| **Service** | Business rules, orchestration, throws NestJS exceptions (`NotFoundException`, `BadRequestException`). |
| **Repository** | Database queries via Prisma. Returns plain objects or Prisma models. No HTTP concerns. |
| **DTO** | Input validation with `class-validator`, output shape with `class-transformer`. |
| **Module** | Wires controller, service, repository. Exports service if other modules need it. |

**Import rule**: Modules must NOT import services from other modules directly. Use module exports/imports via `@Module({ imports, exports })` or shared services in `shared/`.

---

## 4. Coding Rules

1. **Thin controllers**: Max ~15 lines per endpoint. Delegate everything to the service.
2. **Fat services**: All business validation and orchestration here.
3. **Repository when needed**: Start with Prisma calls in the service. Extract to repository when queries exceed ~3 methods or become complex.
4. **Explicit exceptions**: Use NestJS built-in exceptions (`NotFoundException`, `ConflictException`) — never return `{ error: true }` objects.
5. **No cross-module Prisma calls**: A service must only access its own tables. Cross-feature data needs go through the other module's exported service.
6. **Response DTOs**: Map service results to response DTOs in the controller or via a simple mapper function in the module — not mandatory classes unless the team prefers it.

---

## 5. Complete Example: Products Module

### `dto/create-product.dto.ts`
```typescript
import { IsString, IsNumber, Min, MinLength } from 'class-validator';

export class CreateProductDto {
  @IsString()
  @MinLength(1)
  name: string;

  @IsNumber()
  @Min(0.01)
  price: number;
}
```

### `dto/product-response.dto.ts`
```typescript
export class ProductResponseDto {
  id: string;
  name: string;
  price: number;
  createdAt: Date;
}
```

### `products.repository.ts`
```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../shared/prisma/prisma.service';
import { CreateProductDto } from './dto/create-product.dto';

@Injectable()
export class ProductsRepository {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.product.findMany({ orderBy: { createdAt: 'desc' } });
  }

  findById(id: string) {
    return this.prisma.product.findUnique({ where: { id } });
  }

  create(data: CreateProductDto) {
    return this.prisma.product.create({ data });
  }

  delete(id: string) {
    return this.prisma.product.delete({ where: { id } });
  }
}
```

### `products.service.ts`
```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { ProductsRepository } from './products.repository';
import { CreateProductDto } from './dto/create-product.dto';

@Injectable()
export class ProductsService {
  constructor(private readonly repository: ProductsRepository) {}

  async findAll() {
    return this.repository.findAll();
  }

  async findById(id: string) {
    const product = await this.repository.findById(id);
    if (!product) throw new NotFoundException(`Product ${id} not found`);
    return product;
  }

  async create(dto: CreateProductDto) {
    return this.repository.create(dto);
  }

  async delete(id: string) {
    await this.findById(id); // throws if not found
    await this.repository.delete(id);
  }
}
```

### `products.controller.ts`
```typescript
import { Controller, Get, Post, Delete, Param, Body } from '@nestjs/common';
import { ProductsService } from './products.service';
import { CreateProductDto } from './dto/create-product.dto';

@Controller('products')
export class ProductsController {
  constructor(private readonly productsService: ProductsService) {}

  @Get()
  findAll() {
    return this.productsService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.productsService.findById(id);
  }

  @Post()
  create(@Body() dto: CreateProductDto) {
    return this.productsService.create(dto);
  }

  @Delete(':id')
  delete(@Param('id') id: string) {
    return this.productsService.delete(id);
  }
}
```

### `products.module.ts`
```typescript
import { Module } from '@nestjs/common';
import { ProductsController } from './products.controller';
import { ProductsService } from './products.service';
import { ProductsRepository } from './products.repository';

@Module({
  controllers: [ProductsController],
  providers: [ProductsService, ProductsRepository],
  exports: [ProductsService],
})
export class ProductsModule {}
```

### `app.module.ts`
```typescript
import { Module } from '@nestjs/common';
import { ProductsModule } from './modules/products/products.module';
import { PrismaService } from './shared/prisma/prisma.service';

@Module({
  imports: [ProductsModule],
  providers: [PrismaService],
})
export class AppModule {}
```

---

## 6. Cross-Module Communication

When module A needs data from module B:

```typescript
// orders.module.ts
@Module({
  imports: [ProductsModule],  // Import the module, not the repository
  controllers: [OrdersController],
  providers: [OrdersService],
})
export class OrdersModule {}

// orders.service.ts
@Injectable()
export class OrdersService {
  constructor(private readonly productsService: ProductsService) {}

  async createOrder(productId: string) {
    const product = await this.productsService.findById(productId);
    // ... create order with product data
  }
}
```

Never inject `ProductsRepository` from another module — always go through the exported **service**.

---

## 7. Error Handling

Use NestJS built-in HTTP exceptions in services:

```typescript
import { NotFoundException, ConflictException, BadRequestException } from '@nestjs/common';

// In service methods:
if (!user) throw new NotFoundException(`User ${id} not found`);
if (existing) throw new ConflictException('Email already registered');
if (amount <= 0) throw new BadRequestException('Amount must be positive');
```

For global formatting, use a shared filter in `shared/filters/`:

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    response.status(status).json({
      statusCode: status,
      message: exception instanceof HttpException
        ? exception.message
        : 'Internal server error',
      timestamp: new Date().toISOString(),
    });
  }
}
```

---

## 8. When to Use This vs Hexagonal

| Use **Modular** (this skill) | Use **Hexagonal** (`backend-nestjs-vertical-hexagonal`) |
|---|---|
| MVPs, startups, small teams | Enterprise, large teams |
| Fast iteration, idiomatic NestJS | Strict domain isolation, pure TS core |
| Prisma direct access is fine | ORM must not leak into domain |
| Standard NestJS patterns | DDD with ports, adapters, use cases |
