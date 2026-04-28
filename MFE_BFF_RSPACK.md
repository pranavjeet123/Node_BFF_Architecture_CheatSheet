# BFF in Microfrontend Architecture with Rspack

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Team Ownership Model](#team-ownership-model)
- [Rspack Module Federation Setup](#rspack-module-federation-setup)
- [BFF Routing Strategies](#bff-routing-strategies)
- [Shared Auth Across MFEs](#shared-auth-across-mfes)
- [Cross-MFE Data Sharing](#cross-mfe-data-sharing)
- [BFF Contract-First Design](#bff-contract-first-design)
- [Error Handling Consistency](#error-handling-consistency)
- [Observability — Trace Across MFE + BFF](#observability--trace-across-mfe--bff)
- [Best Practices Checklist](#best-practices-checklist)
- [Common Pitfalls](#common-pitfalls)

---

## Architecture Overview

In a Microfrontend (MFE) architecture each independently deployed frontend slice has a matching BFF slice. This preserves **team autonomy end-to-end** — the same team that owns the UI owns the backend contract shaping data for that UI.

### Two Models

#### Model A — Dedicated BFF per MFE (recommended)

```
                    ┌──────────────────────────────────┐
                    │         Shell / Host App          │
                    │  (Rspack Module Federation host)  │
                    └────────┬────────┬────────┬────────┘
                             │        │        │
                     loads   │        │        │  loads remotes
                    ┌────────▼──┐  ┌──▼──────┐  ┌▼──────────┐
                    │ Catalog   │  │  Cart   │  │ Checkout  │
                    │  Remote   │  │ Remote  │  │  Remote   │
                    └─────┬─────┘  └────┬────┘  └─────┬─────┘
                          │             │              │
                          ▼             ▼              ▼
                   [Catalog BFF]  [Cart BFF]   [Checkout BFF]
                          │             │              │
                    ┌─────┴──────┬──────┴──────────────┘
                    │            │            │
             [Product Svc] [Inventory] [Order Svc] [Payment Svc]
```

Each remote MFE talks **only** to its own BFF. No MFE calls another MFE's BFF directly.

#### Model B — Shared BFF with domain routing (simpler, less autonomous)

```
[Shell] + [All Remotes]
         │
         ▼
  [Shared Web BFF]
    /catalog/* → Catalog domain
    /cart/*    → Cart domain
    /checkout/ → Checkout domain
         │
         ▼
  [Microservices]
```

Use Model B when teams are small and Model A's operational overhead isn't justified yet.

---

## Team Ownership Model

The BFF lives in the **same repo and same CI/CD pipeline** as the MFE it serves.

```
teams/
├── catalog/
│   ├── frontend/          ← Rspack remote (catalog MFE)
│   │   ├── rspack.config.ts
│   │   └── src/
│   └── bff/               ← Catalog BFF (Node.js)
│       ├── src/routes/
│       └── Dockerfile
│
├── cart/
│   ├── frontend/          ← Rspack remote (cart MFE)
│   └── bff/               ← Cart BFF
│
├── checkout/
│   ├── frontend/
│   └── bff/
│
└── shell/
    ├── frontend/          ← Rspack host (assembles remotes)
    └── bff/               ← Shell BFF (auth, user session, navigation config)
```

**Rule:** A team deploys their frontend and BFF together. Breaking changes in the BFF are made in the same PR as the frontend that consumes them.

---

## Rspack Module Federation Setup

### Install

```bash
# In each MFE workspace
npm install -D @rspack/core @rspack/cli
npm install -D @module-federation/enhanced   # MF 2.0 runtime
```

### Host (Shell) — `rspack.config.ts`

```typescript
// shell/frontend/rspack.config.ts
import { defineConfig } from '@rspack/cli';
import { ModuleFederationPlugin } from '@module-federation/enhanced/rspack';

export default defineConfig({
  entry: './src/index.ts',
  output: {
    publicPath: 'auto',
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        // In development: point to local dev servers
        // In production: point to CDN-deployed remote entry files
        catalog:  `catalog@${process.env.CATALOG_REMOTE_URL}/remoteEntry.js`,
        cart:     `cart@${process.env.CART_REMOTE_URL}/remoteEntry.js`,
        checkout: `checkout@${process.env.CHECKOUT_REMOTE_URL}/remoteEntry.js`,
      },
      shared: {
        react:        { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom':  { singleton: true, requiredVersion: '^18.0.0' },
        // Share the BFF client config so remotes inherit base URL + auth headers
        '@acme/bff-client': { singleton: true },
      },
    }),
  ],
});
```

### Remote (Catalog MFE) — `rspack.config.ts`

```typescript
// catalog/frontend/rspack.config.ts
import { defineConfig } from '@rspack/cli';
import { ModuleFederationPlugin } from '@module-federation/enhanced/rspack';

export default defineConfig({
  entry: './src/index.ts',
  output: {
    publicPath: 'auto',
    uniqueName: 'catalog',   // required for MF 2.0
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'catalog',
      filename: 'remoteEntry.js',
      exposes: {
        './App':            './src/App',
        './ProductCard':    './src/components/ProductCard',
        './CategoryNav':    './src/components/CategoryNav',
      },
      shared: {
        react:       { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
        '@acme/bff-client': { singleton: true },
      },
    }),
  ],
  devServer: {
    port: 3001,
    headers: { 'Access-Control-Allow-Origin': '*' },
  },
});
```

### Type-Safe Remote Imports in the Shell

```typescript
// shell/frontend/src/App.tsx
import React, { Suspense, lazy } from 'react';

// Rspack/MF generates types automatically with @module-federation/dts-plugin
const CatalogApp  = lazy(() => import('catalog/App'));
const CartApp     = lazy(() => import('cart/App'));
const CheckoutApp = lazy(() => import('checkout/App'));

export default function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/catalog/*"  element={<CatalogApp />} />
        <Route path="/cart/*"     element={<CartApp />} />
        <Route path="/checkout/*" element={<CheckoutApp />} />
      </Routes>
    </Suspense>
  );
}
```

---

## BFF Routing Strategies

### Strategy 1 — Path-Prefix Routing via API Gateway

Each BFF is a separate service. An API Gateway (nginx, Kong, AWS ALB) routes by path prefix.

```nginx
# nginx.conf — route MFE traffic to their BFFs
upstream catalog_bff  { server catalog-bff:3001; }
upstream cart_bff     { server cart-bff:3002; }
upstream checkout_bff { server checkout-bff:3003; }
upstream shell_bff    { server shell-bff:3000; }

server {
  listen 80;

  location /api/catalog/  { proxy_pass http://catalog_bff;  }
  location /api/cart/     { proxy_pass http://cart_bff;     }
  location /api/checkout/ { proxy_pass http://checkout_bff; }
  location /api/          { proxy_pass http://shell_bff;    }

  # All static MFE assets served from CDN — no routing needed here
}
```

### Strategy 2 — Shared BFF Client in the MFE

Ship a shared `@acme/bff-client` package that each MFE uses. It injects the auth header automatically and routes to the correct BFF base URL per environment.

```typescript
// packages/bff-client/src/index.ts
import axios, { AxiosInstance } from 'axios';

interface BffClientOptions {
  domain: 'catalog' | 'cart' | 'checkout' | 'shell';
}

const BASE_URLS: Record<string, string> = {
  catalog:  process.env.CATALOG_BFF_URL  ?? '/api/catalog',
  cart:     process.env.CART_BFF_URL     ?? '/api/cart',
  checkout: process.env.CHECKOUT_BFF_URL ?? '/api/checkout',
  shell:    process.env.SHELL_BFF_URL    ?? '/api',
};

export function createBffClient({ domain }: BffClientOptions): AxiosInstance {
  const client = axios.create({
    baseURL: BASE_URLS[domain],
    withCredentials: true,   // send session cookie for web auth
  });

  // Inject auth token from the shared auth store
  client.interceptors.request.use(config => {
    const token = window.__SHELL_AUTH__?.getToken();
    if (token) config.headers['Authorization'] = `Bearer ${token}`;
    config.headers['X-Request-Id'] = crypto.randomUUID();
    return config;
  });

  return client;
}

// Usage in Catalog MFE
// catalog/frontend/src/api/catalog.api.ts
import { createBffClient } from '@acme/bff-client';

const bff = createBffClient({ domain: 'catalog' });

export const catalogApi = {
  getProducts:   (params) => bff.get('/products', { params }),
  getProduct:    (id)     => bff.get(`/products/${id}`),
  getCategories: ()       => bff.get('/categories'),
};
```

---

## Shared Auth Across MFEs

Auth state must be consistent across all MFEs loaded in the shell. The Shell BFF owns the auth session; remotes consume it via a shared singleton.

### Shell BFF — Session Endpoint

```typescript
// shell/bff/src/routes/auth.route.ts
import { Router } from 'express';
import { authenticate } from '../middleware/auth';

const router = Router();

// Called by the shell on boot — returns user context for all MFEs
router.get('/session', authenticate, (req, res) => {
  res.json({
    user: {
      id:     req.user.id,
      email:  req.user.email,
      name:   req.user.name,
      roles:  req.user.roles,
      avatar: req.user.avatarUrl,
    },
    permissions: derivePermissions(req.user.roles),
    featureFlags: req.featureFlags,  // evaluated once, shared with all MFEs
  });
});

router.post('/logout', (req, res) => {
  req.session.destroy(() => {
    res.clearCookie('session').json({ ok: true });
  });
});

export default router;
```

### Shared Auth Store (MF singleton)

```typescript
// packages/auth-store/src/index.ts
// This is shared via Module Federation — loaded once by the shell,
// consumed by all remotes as a singleton.

interface AuthState {
  user: User | null;
  token: string | null;
  permissions: string[];
  featureFlags: Record<string, boolean>;
}

class AuthStore {
  private state: AuthState = { user: null, token: null, permissions: [], featureFlags: {} };
  private listeners = new Set<() => void>();

  async init() {
    const res = await fetch('/api/session', { credentials: 'include' });
    if (res.ok) {
      const data = await res.json();
      this.state = {
        user: data.user,
        token: data.token ?? null,
        permissions: data.permissions,
        featureFlags: data.featureFlags,
      };
      this.notify();
    }
  }

  getUser()         { return this.state.user; }
  getToken()        { return this.state.token; }
  hasPermission(p)  { return this.state.permissions.includes(p); }
  getFlag(key)      { return this.state.featureFlags[key] ?? false; }

  subscribe(fn: () => void) {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
  }

  private notify() { this.listeners.forEach(fn => fn()); }
}

// Exported as singleton — MF ensures only one instance exists
export const authStore = new AuthStore();

// React hook for MFEs
export function useAuth() {
  const [user, setUser] = React.useState(authStore.getUser());
  React.useEffect(() => authStore.subscribe(() => setUser(authStore.getUser())), []);
  return { user, hasPermission: authStore.hasPermission.bind(authStore) };
}
```

```typescript
// shell/frontend/rspack.config.ts — expose auth store as MF singleton
exposes: {
  './auth': './src/auth-store/index',
},
shared: {
  '@acme/auth-store': { singleton: true, eager: true },
},

// Catalog remote — consume from shell
import { useAuth } from 'shell/auth';
```

---

## Cross-MFE Data Sharing

MFEs must not call each other's BFFs directly. When MFE A needs data owned by MFE B, there are two clean approaches:

### Approach 1 — Custom Events (loose coupling)

```typescript
// cart/frontend/src/Cart.tsx — emit event after add to cart
function addToCart(productId: string) {
  cartApi.addItem(productId).then(cart => {
    // Dispatch a DOM event — shell or other MFEs can listen
    window.dispatchEvent(new CustomEvent('cart:updated', {
      detail: { count: cart.itemCount, total: cart.total },
      bubbles: true,
    }));
  });
}

// shell/frontend/src/Header.tsx — react to cart events
React.useEffect(() => {
  const handler = (e: CustomEvent) => setCartCount(e.detail.count);
  window.addEventListener('cart:updated', handler);
  return () => window.removeEventListener('cart:updated', handler);
}, []);
```

### Approach 2 — Shared State via MF Singleton

```typescript
// packages/shared-state/src/cart-state.ts
// Exposed by the cart remote, shared as MF singleton

class CartState {
  private count = 0;
  private listeners = new Set<(count: number) => void>();

  setCount(n: number) {
    this.count = n;
    this.listeners.forEach(fn => fn(n));
  }

  subscribe(fn: (count: number) => void) {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
  }
}

export const cartState = new CartState();
```

---

## BFF Contract-First Design

Each BFF team publishes a **typed contract** (OpenAPI or Zod schema) that the MFE team imports. This makes the frontend–BFF boundary explicit and versioned.

### Zod Contract Package

```
packages/
└── contracts/
    ├── catalog-contract/   ← owned by catalog team
    │   ├── src/
    │   │   ├── product.schema.ts
    │   │   └── index.ts
    │   └── package.json
    └── cart-contract/
        └── src/
            ├── cart.schema.ts
            └── index.ts
```

```typescript
// packages/contracts/catalog-contract/src/product.schema.ts
import { z } from 'zod';

export const ProductSchema = z.object({
  id:          z.string().uuid(),
  name:        z.string(),
  slug:        z.string(),
  price:       z.number().positive(),
  currency:    z.string().length(3),
  imageUrl:    z.string().url(),
  inStock:     z.boolean(),
  rating:      z.number().min(0).max(5).nullable(),
});

export const ProductListSchema = z.object({
  items:       z.array(ProductSchema),
  total:       z.number().int(),
  page:        z.number().int(),
  pageSize:    z.number().int(),
});

// Infer TypeScript types from schema — single source of truth
export type Product     = z.infer<typeof ProductSchema>;
export type ProductList = z.infer<typeof ProductListSchema>;
```

```typescript
// catalog/bff/src/routes/products.route.ts — validate outgoing response
import { ProductListSchema } from '@acme/catalog-contract';

router.get('/products', async (req, res, next) => {
  try {
    const raw = await productService.list(req.query);
    const shaped = raw.items.map(toClientProduct);

    // Validate your own response before sending — catch contract drift early
    const validated = ProductListSchema.parse({
      items: shaped,
      total: raw.total,
      page: Number(req.query.page ?? 1),
      pageSize: Number(req.query.pageSize ?? 20),
    });

    res.json(validated);
  } catch (err) {
    next(err);
  }
});
```

```typescript
// catalog/frontend/src/api/catalog.api.ts — validate incoming response
import { ProductListSchema, type ProductList } from '@acme/catalog-contract';

export async function getProducts(params: SearchParams): Promise<ProductList> {
  const { data } = await bff.get('/products', { params });

  // Parse validates shape at runtime — surfaces BFF contract drift instantly
  return ProductListSchema.parse(data);
}
```

---

## Error Handling Consistency

All BFFs must return the **same error envelope**. The shell and MFEs share one error-parsing utility.

```typescript
// packages/bff-client/src/error.ts — shared error shape

export interface BffError {
  message:   string;
  code:      string;
  requestId: string;
  field?:    string;   // for validation errors
}

export class BffRequestError extends Error {
  constructor(
    public status: number,
    public error: BffError,
  ) {
    super(error.message);
    this.name = 'BffRequestError';
  }
}

// Axios response interceptor — applied in createBffClient
client.interceptors.response.use(
  res => res,
  err => {
    const status  = err.response?.status ?? 503;
    const payload = err.response?.data?.error ?? {
      message:   'Unexpected error',
      code:      'UNKNOWN',
      requestId: '',
    };
    return Promise.reject(new BffRequestError(status, payload));
  }
);
```

```typescript
// Every BFF (catalog, cart, checkout) registers the same error middleware
// packages/bff-utils/src/error-handler.ts

import { Request, Response, NextFunction } from 'express';
import { ZodError } from 'zod';
import { logger } from './logger';

export function errorHandler(err: any, req: Request, res: Response, _next: NextFunction) {
  if (err instanceof ZodError) {
    return res.status(422).json({
      error: {
        message:   'Validation failed',
        code:      'VALIDATION_ERROR',
        requestId: req.id,
        fields:    err.errors.map(e => ({ path: e.path.join('.'), message: e.message })),
      },
    });
  }

  const status  = err.status ?? 500;
  const message = status < 500 ? err.message : 'An unexpected error occurred';

  logger.error({ err, path: req.path, requestId: req.id }, 'BFF error');

  res.status(status).json({
    error: { message, code: err.code ?? 'INTERNAL_ERROR', requestId: req.id },
  });
}
```

---

## Observability — Trace Across MFE + BFF

Distributed tracing must span the browser → BFF → microservice call chain.

### Generate a Trace ID in the Shell and Pass It Down

```typescript
// shell/frontend/src/tracing.ts
export function getRootTraceId(): string {
  let id = sessionStorage.getItem('traceId');
  if (!id) {
    id = crypto.randomUUID();
    sessionStorage.setItem('traceId', id);
  }
  return id;
}

// Inject into every BFF request
client.interceptors.request.use(config => {
  config.headers['X-Trace-Id']   = getRootTraceId();
  config.headers['X-Span-Id']    = crypto.randomUUID();
  config.headers['X-Request-Id'] = crypto.randomUUID();
  return config;
});
```

```typescript
// Any BFF — thread trace ID through to downstream calls
app.use((req, _res, next) => {
  req.traceId   = req.headers['x-trace-id'] as string ?? randomUUID();
  req.requestId = req.headers['x-request-id'] as string ?? randomUUID();
  next();
});

// In service calls — propagate headers
const { data } = await productClient.get('/products', {
  headers: {
    'X-Trace-Id':    req.traceId,
    'X-Request-Id':  randomUUID(),   // new span ID for this hop
  },
});
```

### Structured Logging with Context

```typescript
// packages/bff-utils/src/logger.ts
import pino from 'pino';

export const logger = pino({ level: process.env.LOG_LEVEL ?? 'info' });

// Request-scoped child logger — attach to req in middleware
app.use((req, _res, next) => {
  req.log = logger.child({
    requestId: req.requestId,
    traceId:   req.traceId,
    mfe:       req.headers['x-mfe-name'],   // which MFE made the call
    path:      req.path,
    method:    req.method,
  });
  next();
});

// In route handlers
req.log.info({ userId: req.user.id }, 'Fetching product list');
req.log.error({ err }, 'Product service call failed');
```

---

## Best Practices Checklist

### Architecture
- [ ] Each MFE team owns its BFF end-to-end — same repo, same CI pipeline
- [ ] MFEs never call each other's BFFs directly
- [ ] The Shell BFF owns auth/session; remotes consume auth via a shared singleton
- [ ] API Gateway handles routing to BFFs by path prefix — not the BFFs themselves
- [ ] Contracts (Zod/OpenAPI) are published as packages shared between BFF and MFE

### Rspack / Module Federation
- [ ] `singleton: true` on all shared packages (React, auth store, BFF client)
- [ ] `eager: true` only on the auth store — avoid it elsewhere (blocks initial parse)
- [ ] `uniqueName` set on every remote `rspack.config.ts`
- [ ] `publicPath: 'auto'` so remotes work from any CDN URL
- [ ] Use `@module-federation/dts-plugin` to share TypeScript types across remotes
- [ ] Remote entry URLs come from environment variables — no hardcoded URLs

### BFF Design
- [ ] Validate outgoing BFF responses against the published contract (Zod parse before `res.json`)
- [ ] Validate incoming request bodies at every BFF route
- [ ] All BFFs use the same error envelope shape — shared middleware from a utility package
- [ ] Set timeouts on every downstream service call
- [ ] Fan out independent service calls with `Promise.all` or `Promise.allSettled`
- [ ] Strip internal/sensitive fields in transformer functions — never rely on "not being requested"

### Observability
- [ ] `X-Request-Id` generated per request and returned in every error response
- [ ] `X-Trace-Id` propagated from browser through BFF to downstream services
- [ ] `x-mfe-name` header sent by every MFE so BFF logs identify the calling MFE
- [ ] Structured JSON logging (Pino) in all BFFs
- [ ] BFF metrics: p50/p95 latency, error rate, downstream call count — per MFE + per route

### Security
- [ ] CORS in each BFF restricted to your shell origin — not `*`
- [ ] The shell BFF is the only service that reads and validates the user's session cookie
- [ ] Remote BFFs accept only the internal `X-Internal-Token` header from the API gateway — never direct internet traffic
- [ ] Helmet middleware on all BFFs
- [ ] Rate limiting keyed on `userId` (not IP) to avoid affecting shared NAT users

---

## Common Pitfalls

| Pitfall | What Goes Wrong | Fix |
|---|---|---|
| **Shared BFF for multiple MFEs** | Teams block each other; data shapes become generic | One BFF per MFE team |
| **MFE calls sibling MFE's BFF** | Tight coupling, shared failure domain | Use custom events or a shared state singleton instead |
| **Auth token managed in each MFE** | Token refresh races, inconsistent expiry | Shell owns one auth store, shared as MF singleton |
| **No `singleton: true` on shared libs** | Multiple React instances → hook errors, context breaks | Always `singleton: true` for React, ReactDOM, auth store |
| **Hardcoded remote entry URLs** | Cannot deploy to different environments | Use env vars for all remote entry URLs |
| **Contract drift (BFF changes silently)** | MFE crashes at runtime | Validate BFF response against Zod schema before sending |
| **Different error shapes per BFF** | Each MFE needs custom error parsing | Share one `errorHandler` middleware package |
| **No trace ID propagation** | Cannot correlate a browser error to a BFF log | Generate trace ID in shell, pass via headers through every hop |
| **`eager: true` on large shared libs** | Increases shell initial parse time | Use `eager` only for tiny critical singletons (auth store) |
| **BFF deployed separately from MFE** | Contract changes break live MFE before both are updated | Deploy MFE and its BFF atomically in the same pipeline stage |
