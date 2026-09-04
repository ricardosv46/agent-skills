---
name: frontend-react-modular
description: Guidelines for building clean, modular React apps with Vertical Slicing and Screaming Architecture. Uses hooks for logic, presentational components with props only, and feature-first modules. Supports Vite and Next.js. Use when the user wants modular frontend architecture without hexagonal/DDD overhead, or when preparing for future React Native migration.
---

# React Modular Architecture (Vertical Slice + Clean Code)

Pragmatic **Feature-First / Vertical Slice** architecture for React apps. No hexagonal layers, no ports & adapters, no ESLint architecture plugin — just clean, reusable, module-based code.

---

## 1. Core Principles

1. **Screaming Architecture**: Folder names reflect business features (`products`, `auth`, `billing`), not technical layers (`controllers`, `utils`).
2. **Hooks Own Logic**: All business logic, data fetching, form state, and side effects live in custom hooks — never inside presentational components.
3. **Components Are Dumb**: UI components receive data and callbacks via **props only**. No direct API calls, no global state access inside leaf components.
4. **One Module = One Feature**: Each feature is self-contained under `modules/{feature}/` with its own types, API, hooks, components, and views.
5. **UI Decoupled from Platform**: Keep styling and layout in components; keep logic in hooks and `api.ts`. This makes a future React Native migration feasible — hooks and API layers reuse as-is.

---

## 2. Directory Layout

### Vite
```text
src/
├── main.tsx
├── routes.tsx                          # Composition root (wire modules to routes)
├── shared/
│   ├── ui/                             # Reusable design primitives (Button, Input, Card)
│   ├── hooks/                          # Generic hooks (useDebounce, useLocalStorage)
│   └── lib/
│       └── httpClient.ts               # Axios/fetch instance with interceptors
└── modules/
    └── products/
        ├── types.ts                    # Product, CreateProductInput
        ├── api.ts                      # getProducts(), createProduct()
        ├── hooks/
        │   ├── useProducts.ts          # List: fetch, loading, error, refresh
        │   └── useProductForm.ts       # Form: validation, submit, reset
        ├── components/
        │   ├── ProductCard.tsx         # Props only: { product, onDelete }
        │   └── ProductList.tsx         # Props only: { products, onDelete }
        └── views/
            └── ProductsView.tsx        # Composes hooks + components
```

### Next.js App Router
```text
src/
├── app/                                # Thin routing shells only
│   └── products/
│       └── page.tsx                    # Imports ProductsView
├── shared/
└── modules/
    └── products/
        ├── types.ts
        ├── api.ts
        ├── hooks/
        ├── components/
        └── views/
            └── ProductsView.tsx        # "use client"
```

---

## 3. Layer Responsibilities

| Layer | Responsibility | Can Import From |
|-------|---------------|-----------------|
| `types.ts` | Interfaces, type aliases, constants | Nothing (pure TS) |
| `api.ts` | HTTP calls, response mapping | `types`, `shared/lib` |
| `hooks/` | State, side effects, orchestration | `types`, `api`, other hooks |
| `components/` | Render UI from props | `types`, `shared/ui` |
| `views/` | Compose hooks + components | All of the above in same module |
| `shared/` | Cross-feature reusables | Only `shared/` |

**Import rule**: Modules must NOT import from other modules directly. Share via `shared/` or pass data through composition roots (`routes.tsx`, `page.tsx`).

---

## 4. Coding Rules

1. **No barrel files (`index.ts`)**: Import directly from source files to preserve tree-shaking.
2. **No logic in leaf components**: If a component needs `useState`, `useEffect`, or API calls, extract a hook.
3. **Props over context for local state**: Use React Context only for truly global concerns (auth session, theme). Feature state stays in hooks within the module.
4. **Colocate validations**: Keep form validation inside the hook (`useProductForm`) or a small `validators.ts` in the same module — not scattered across components.
5. **Server state**: Prefer React Query / TanStack Query in hooks for caching, refetch, and deduplication.
6. **Client state**: Use `useState`/`useReducer` in hooks. Zustand only when state must be shared across distant components in the same module.

---

## 5. Complete Example: Products Module

### `types.ts`
```typescript
export interface Product {
  id: string;
  name: string;
  price: number;
}

export interface CreateProductInput {
  name: string;
  price: number;
}
```

### `api.ts`
```typescript
import { httpClient } from '../../../shared/lib/httpClient';
import { Product, CreateProductInput } from './types';

export async function getProducts(): Promise<Product[]> {
  const { data } = await httpClient.get<Product[]>('/products');
  return data;
}

export async function createProduct(input: CreateProductInput): Promise<Product> {
  const { data } = await httpClient.post<Product>('/products', input);
  return data;
}

export async function deleteProduct(id: string): Promise<void> {
  await httpClient.delete(`/products/${id}`);
}
```

### `hooks/useProducts.ts`
```typescript
import { useState, useEffect, useCallback } from 'react';
import { Product } from '../types';
import * as productsApi from '../api';

export function useProducts() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchProducts = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);
      const data = await productsApi.getProducts();
      setProducts(data);
    } catch (err: any) {
      setError(err.message ?? 'Error loading products');
    } finally {
      setLoading(false);
    }
  }, []);

  const removeProduct = useCallback(async (id: string) => {
    await productsApi.deleteProduct(id);
    await fetchProducts();
  }, [fetchProducts]);

  useEffect(() => { fetchProducts(); }, [fetchProducts]);

  return { products, loading, error, refresh: fetchProducts, removeProduct };
}
```

### `hooks/useProductForm.ts`
```typescript
import { useState } from 'react';
import * as productsApi from '../api';

export function useProductForm(onSuccess?: () => void) {
  const [name, setName] = useState('');
  const [price, setPrice] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const parsedPrice = Number(price);
    if (!name.trim() || parsedPrice <= 0) {
      setError('Name and valid price are required');
      return;
    }
    try {
      setSubmitting(true);
      setError(null);
      await productsApi.createProduct({ name: name.trim(), price: parsedPrice });
      setName('');
      setPrice('');
      onSuccess?.();
    } catch (err: any) {
      setError(err.message ?? 'Failed to create product');
    } finally {
      setSubmitting(false);
    }
  };

  return { name, setName, price, setPrice, error, submitting, handleSubmit };
}
```

### `components/ProductCard.tsx`
```tsx
import { Product } from '../types';

interface ProductCardProps {
  product: Product;
  onDelete: (id: string) => void;
}

export function ProductCard({ product, onDelete }: ProductCardProps) {
  return (
    <div className="product-card">
      <span>{product.name}</span>
      <span>${product.price}</span>
      <button onClick={() => onDelete(product.id)}>Delete</button>
    </div>
  );
}
```

### `components/ProductList.tsx`
```tsx
import { Product } from '../types';
import { ProductCard } from './ProductCard';

interface ProductListProps {
  products: Product[];
  onDelete: (id: string) => void;
}

export function ProductList({ products, onDelete }: ProductListProps) {
  if (products.length === 0) return <p>No products yet.</p>;

  return (
    <div className="product-list">
      {products.map((product) => (
        <ProductCard key={product.id} product={product} onDelete={onDelete} />
      ))}
    </div>
  );
}
```

### `views/ProductsView.tsx`
```tsx
'use client';

import { useProducts } from '../hooks/useProducts';
import { useProductForm } from '../hooks/useProductForm';
import { ProductList } from '../components/ProductList';

export function ProductsView() {
  const { products, loading, error, refresh, removeProduct } = useProducts();
  const form = useProductForm(refresh);

  if (loading) return <div>Loading...</div>;
  if (error) return <div style={{ color: 'red' }}>{error}</div>;

  return (
    <div>
      <h1>Products</h1>
      <form onSubmit={form.handleSubmit}>
        <input value={form.name} onChange={(e) => form.setName(e.target.value)} placeholder="Name" />
        <input value={form.price} onChange={(e) => form.setPrice(e.target.value)} placeholder="Price" type="number" />
        <button type="submit" disabled={form.submitting}>Add</button>
        {form.error && <span style={{ color: 'red' }}>{form.error}</span>}
      </form>
      <ProductList products={products} onDelete={removeProduct} />
    </div>
  );
}
```

---

## 6. Composition Roots

### Vite `routes.tsx`
```tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { ProductsView } from './modules/products/views/ProductsView';

export function AppRoutes() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/products" element={<ProductsView />} />
      </Routes>
    </BrowserRouter>
  );
}
```

### Next.js `app/products/page.tsx`
```tsx
import { ProductsView } from '@/modules/products/views/ProductsView';

export default function ProductsPage() {
  return <ProductsView />;
}
```

---

## 7. React Native Readiness

When migrating to React Native later:

| Layer | Reuse |
|-------|-------|
| `types.ts` | 100% — copy as-is |
| `api.ts` | 100% — copy as-is |
| `hooks/` | ~95% — minor adjustments (no DOM APIs) |
| `components/` | Rewrite JSX (`View`, `Text`, `Pressable`) but **same props interface** |
| `views/` | Rewrite layout, keep same hook composition |

**Tip**: Define props interfaces explicitly (`ProductCardProps`) so RN components can implement the same contract.

---

## 8. When to Use This vs Hexagonal

| Use **Modular** (this skill) | Use **Hexagonal** (`frontend-react-vertical-hexagonal`) |
|---|---|
| MVPs, startups, small teams | Enterprise, large teams |
| Fast iteration | Strict domain isolation |
| React Native planned but not immediate | Core logic must be 100% framework-agnostic |
| Pragmatic clean code | Full DDD with ports & adapters |
