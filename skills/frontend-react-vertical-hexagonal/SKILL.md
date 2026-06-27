---
name: frontend-react-vertical-hexagonal
description: Guidelines for implementing Hexagonal Architecture, Vertical Slicing, and Domain-Driven Design (DDD) in React frontends (Vite and Next.js).
---

# Enterprise React (Vite & Next.js) Vertical-Hexagonal & DDD

This guideline defines the industry-standard implementation of **Hexagonal Architecture**, **Vertical Slicing (Feature-First)**, and **Domain-Driven Design (DDD)** in enterprise React applications, supporting both client-side **Vite** and server-client **Next.js App Router** environments.

---

## 1. Core Architectural Pillars

1.  **Strict Isolation of the Core**: The core business logic (`domain/` and `application/`) is written in pure TypeScript. It **MUST NOT** import React, custom hooks, HTML elements, CSS, or libraries like Axios or React Query.
2.  **Use Hooks as Bridging Adapters**: Custom hooks (e.g., `useUserSession`) act as the driving adapters that connect React's reactive UI layer to pure, stateless use cases.
3.  **Ports & Adapters for APIs**: Components and hooks never invoke `fetch` or `axios` directly. They invoke Use Cases that interact with a Repository Port (interface). The HTTP adapter implementing that interface handles request execution.
4.  **Pragmatic DDD**: Keep domain models valid using Value Objects with self-validation rules, preventing invalid states on form submission or local rendering.

---

## 2. Directory Layouts

### A. Vite Client-Side Structure
```text
src/
├── main.tsx
├── routes.tsx                          # App routing composition root
├── shared/                             # Shared Kernel (primitives)
│   ├── domain/
│   │   ├── building-blocks/            # Shared DDD Base Classes
│   │   │   ├── Entity.ts
│   │   │   └── ValueObject.ts
│   │   └── value-objects/
│   │       └── Email.ts
│   └── infrastructure/
│       └── http/
│           └── httpClient.ts
└── modules/
    └── products/                       # Bounded Context / Vertical Slice
        ├── domain/                     # Entities & Repository Ports
        │   ├── Product.ts
        │   └── ProductRepository.ts    # Interface (Port)
        ├── application/                # Pure use cases (Functions/Classes)
        │   └── GetProductsUseCase.ts
        ├── infrastructure/             # Adapters (Axios, LocalStorage, Zustand)
        │   ├── AxiosProductRepository.ts
        │   └── productStore.ts         # Zustand slice for products state
        └── ui/                         # React UI (HTML/CSS/Hooks/Context)
            ├── components/
            │   ├── ProductCard.tsx
            │   └── ProductList.tsx
            ├── hooks/
            │   └── useProducts.ts      # Custom hook bridging core and UI
            └── views/
                └── ProductsView.tsx
```

### B. Next.js App Router Structure
In Next.js, the `src/app/` directory acts as a thin routing and composition shell. Use-case and adapter instantiation occurs on the Server Page, injecting repositories into Client Components.
```text
src/
├── app/                                # Next.js App Router (Routing Shell)
│   ├── layout.tsx
│   └── products/
│       └── page.tsx                    # Server Page instantiating repository
├── shared/
└── modules/
    └── products/
        ├── domain/
        ├── application/
        ├── infrastructure/
        └── ui/
            ├── hooks/
            │   └── useProducts.ts      # Marked with 'use client'
            └── views/
                └── ProductsView.tsx    # Client Component View
```

---

## 3. DDD Base Primitives

Define base classes in `src/shared/domain/building-blocks/` to maintain structural consistency.

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

---

## 4. Real-World Implementation Flow

### A. Domain Layer: Entity & Repository Port
`src/modules/products/domain/Product.ts`
```typescript
import { Entity } from '../../../shared/domain/building-blocks/Entity';

interface ProductProps {
  name: string;
  price: number;
}

export class Product extends Entity<ProductProps> {
  constructor(props: ProductProps, id?: string) {
    if (props.price < 0) throw new Error('Price cannot be negative');
    super(props, id);
  }

  get name(): string { return this.props.name; }
  get price(): number { return this.props.price; }
}
```

`src/modules/products/domain/ProductRepository.ts`
```typescript
import { Product } from './Product';

export interface ProductRepository {
  getAll(): Promise<Product[]>;
  save(product: Product): Promise<void>;
}
```

### B. Application Layer: Use Case
`src/modules/products/application/GetProductsUseCase.ts`
```typescript
import { Product } from '../domain/Product';
import { ProductRepository } from '../domain/ProductRepository';

export class GetProductsUseCase {
  constructor(private readonly productRepository: ProductRepository) {}

  async execute(): Promise<Product[]> {
    return await this.productRepository.getAll();
  }
}
```

### C. Infrastructure Layer: Repository Adapter
`src/modules/products/infrastructure/AxiosProductRepository.ts`
```typescript
import axios from 'axios';
import { Product } from '../domain/Product';
import { ProductRepository } from '../domain/ProductRepository';

interface ApiProduct {
  id: string;
  name: string;
  price: number;
}

export class AxiosProductRepository implements ProductRepository {
  async getAll(): Promise<Product[]> {
    const response = await axios.get<ApiProduct[]>('/api/products');
    return response.data.map(
      (raw) => new Product({ name: raw.name, price: raw.price }, raw.id)
    );
  }

  async save(product: Product): Promise<void> {
    await axios.post('/api/products', {
      id: product.id,
      name: product.name,
      price: product.price,
    });
  }
}
```

### D. UI Layer: React Custom Bridging Hook (Driving Adapter)
The hook manages state, triggers loading indicators, handles errors, and executes Use Cases.
`src/modules/products/ui/hooks/useProducts.ts`
```typescript
import { useState, useEffect } from 'react';
import { Product } from '../../domain/Product';
import { ProductRepository } from '../../domain/ProductRepository';
import { GetProductsUseCase } from '../../application/GetProductsUseCase';

export function useProducts(repository: ProductRepository) {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchProducts = async () => {
    try {
      setLoading(true);
      setError(null);
      const useCase = new GetProductsUseCase(repository);
      const list = await useCase.execute();
      setProducts(list);
    } catch (err: any) {
      setError(err.message || 'Error fetching products');
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchProducts();
  }, []);

  return { products, loading, error, refresh: fetchProducts };
}
```

### E. UI Layer: TSX Component & View
`src/modules/products/ui/views/ProductsView.tsx`
```tsx
import React from 'react';
import { ProductRepository } from '../../domain/ProductRepository';
import { useProducts } from '../hooks/useProducts';

export function ProductsView({ repository }: { repository: ProductRepository }) {
  const { products, loading, error } = useProducts(repository);

  if (loading) return <div>Loading Catalog...</div>;
  if (error) return <div style={{ color: 'red' }}>Error: {error}</div>;

  return (
    <div>
      <h1>🛍️ Product Catalog</h1>
      <ul>
        {products.map((product) => (
          <li key={product.id}>
            {product.name} - ${product.price}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 5. Composition Roots (Wiring Everything Up)

### Vite Routing Shell (`src/routes.tsx`)
```tsx
import { AxiosProductRepository } from './modules/products/infrastructure/AxiosProductRepository';
import { ProductsView } from './modules/products/ui/views/ProductsView';

const productRepository = new AxiosProductRepository();

export function AppRoutes() {
  return (
    <ProductsView repository={productRepository} />
  );
}
```

### Next.js App Router Shell (`src/app/products/page.tsx`)
```tsx
import { AxiosProductRepository } from '@/modules/products/infrastructure/AxiosProductRepository';
import { ProductsView } from '@/modules/products/ui/views/ProductsView';

export default function ProductsPage() {
  // Instantiated on Server Side
  const repository = new AxiosProductRepository();

  return <ProductsView repository={repository} />;
}
```
