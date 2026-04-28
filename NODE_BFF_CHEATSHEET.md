# Node.js BFF Cheatsheet — Code Snippets, Patterns & Best Practices

## Table of Contents
- [Project Setup](#project-setup)
- [Core Patterns](#core-patterns)
  - [Service Aggregation](#1-service-aggregation)
  - [Request Forwarding / Proxy](#2-request-forwarding--proxy)
  - [Response Shaping](#3-response-shaping)
  - [Parallel Fan-Out](#4-parallel-fan-out)
  - [Circuit Breaker](#5-circuit-breaker)
  - [Caching Layer](#6-caching-layer)
  - [Auth Middleware](#7-auth-middleware)
  - [Rate Limiting](#8-rate-limiting)
  - [Error Normalization](#9-error-normalization)
  - [GraphQL BFF](#10-graphql-bff)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)
- [Folder Structure](#folder-structure)

---

## Project Setup

### Minimal Express BFF
```bash
npm init -y
npm install express axios dotenv helmet cors express-rate-limit pino pino-http
npm install -D typescript @types/node @types/express ts-node nodemon
```

**tsconfig.json**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true
  }
}
```

### Fastify BFF (recommended for performance)
```bash
npm install fastify @fastify/cors @fastify/helmet @fastify/rate-limit axios undici
```

---

## Core Patterns

### 1. Service Aggregation

Combine data from multiple downstream services into one response.

```typescript
// src/routes/dashboard.ts
import { Router } from 'express';
import { userService } from '../services/user.service';
import { orderService } from '../services/order.service';
import { analyticsService } from '../services/analytics.service';

const router = Router();

router.get('/dashboard', async (req, res, next) => {
  try {
    const userId = req.user.id;

    // Fan out in parallel — never call services serially if data is independent
    const [user, recentOrders, stats] = await Promise.all([
      userService.getProfile(userId),
      orderService.getRecent(userId, { limit: 5 }),
      analyticsService.getSummary(userId),
    ]);

    // Shape the response for exactly what the dashboard screen needs
    res.json({
      greeting: `Hello, ${user.firstName}`,
      avatar: user.avatarUrl,
      orderCount: recentOrders.total,
      orders: recentOrders.items.map(o => ({
        id: o.id,
        status: o.status,
        total: o.total,
        date: o.createdAt,
      })),
      metrics: {
        spent: stats.totalSpent,
        saved: stats.totalSaved,
      },
    });
  } catch (err) {
    next(err);
  }
});

export default router;
```

---

### 2. Request Forwarding / Proxy

Forward a request to a downstream service, adding auth headers.

```typescript
// src/services/http-client.ts
import axios, { AxiosInstance } from 'axios';

export function createServiceClient(baseURL: string): AxiosInstance {
  const client = axios.create({
    baseURL,
    timeout: 5000,
    headers: { 'Content-Type': 'application/json' },
  });

  // Attach internal service auth token on every request
  client.interceptors.request.use(config => {
    config.headers['X-Internal-Token'] = process.env.INTERNAL_SERVICE_TOKEN!;
    return config;
  });

  // Normalize downstream errors
  client.interceptors.response.use(
    res => res,
    err => {
      const status = err.response?.status ?? 503;
      const message = err.response?.data?.message ?? 'Downstream service error';
      const error = new Error(message) as any;
      error.status = status;
      return Promise.reject(error);
    }
  );

  return client;
}

// Usage
export const userClient = createServiceClient(process.env.USER_SERVICE_URL!);
export const orderClient = createServiceClient(process.env.ORDER_SERVICE_URL!);
```

---

### 3. Response Shaping

Strip internal fields, rename keys, and flatten nested data for the client.

```typescript
// src/transformers/order.transformer.ts

interface RawOrder {
  order_id: string;
  created_at: string;
  line_items: Array<{ sku: string; qty: number; unit_price: number }>;
  customer_internal_ref: string; // never expose to client
  total_cents: number;
}

export interface ClientOrder {
  id: string;
  date: string;
  items: Array<{ sku: string; quantity: number; price: number }>;
  total: number;
}

export function toClientOrder(raw: RawOrder): ClientOrder {
  return {
    id: raw.order_id,
    date: raw.created_at,
    items: raw.line_items.map(item => ({
      sku: item.sku,
      quantity: item.qty,
      price: item.unit_price / 100,
    })),
    total: raw.total_cents / 100,
    // customer_internal_ref intentionally omitted
  };
}
```

---

### 4. Parallel Fan-Out

The most common BFF performance pattern — never await in series when calls are independent.

```typescript
// Bad — serial: takes sum of all latencies
const user   = await userService.get(id);       // 80ms
const orders = await orderService.list(id);     // 120ms
const prefs  = await prefsService.get(id);      // 60ms
// Total: ~260ms

// Good — parallel: takes latency of the slowest
const [user, orders, prefs] = await Promise.all([
  userService.get(id),       // \
  orderService.list(id),     //  > all in-flight simultaneously
  prefsService.get(id),      // /
]);
// Total: ~120ms

// Good — parallel with individual error handling (partial success)
const results = await Promise.allSettled([
  userService.get(id),
  orderService.list(id),
  prefsService.get(id),
]);

const user   = results[0].status === 'fulfilled' ? results[0].value : null;
const orders = results[1].status === 'fulfilled' ? results[1].value : [];
const prefs  = results[2].status === 'fulfilled' ? results[2].value : defaultPrefs;
```

---

### 5. Circuit Breaker

Prevent cascading failures when a downstream service degrades.

```typescript
// npm install opossum
import CircuitBreaker from 'opossum';
import { orderClient } from './http-client';

const options = {
  timeout: 3000,          // call fails if >3s
  errorThresholdPercentage: 50, // open circuit if >50% of calls fail
  resetTimeout: 10000,    // try again after 10s
};

const fetchOrders = (userId: string) =>
  orderClient.get(`/orders?userId=${userId}`).then(r => r.data);

export const orderBreaker = new CircuitBreaker(fetchOrders, options);

orderBreaker.fallback((userId: string) => {
  // Return stale cache or empty default — never throw to the client
  return { items: [], total: 0, stale: true };
});

orderBreaker.on('open',    () => console.warn('Order service circuit OPEN'));
orderBreaker.on('close',   () => console.info('Order service circuit CLOSED'));
orderBreaker.on('halfOpen',() => console.info('Order service circuit HALF-OPEN'));

// Usage
const orders = await orderBreaker.fire(userId);
```

---

### 6. Caching Layer

Cache aggregated responses to reduce downstream load and latency.

```typescript
// src/cache/redis-cache.ts
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

export async function cached<T>(
  key: string,
  ttlSeconds: number,
  fn: () => Promise<T>
): Promise<T> {
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit) as T;

  const value = await fn();
  await redis.setEx(key, ttlSeconds, JSON.stringify(value));
  return value;
}

// Usage in a route
router.get('/profile/:id', async (req, res, next) => {
  try {
    const profile = await cached(
      `profile:${req.params.id}`,
      300, // 5 min TTL
      () => userService.getProfile(req.params.id)
    );
    res.json(profile);
  } catch (err) {
    next(err);
  }
});

// Cache invalidation on mutation
router.put('/profile/:id', async (req, res, next) => {
  try {
    const updated = await userService.updateProfile(req.params.id, req.body);
    await redis.del(`profile:${req.params.id}`);
    res.json(updated);
  } catch (err) {
    next(err);
  }
});
```

---

### 7. Auth Middleware

Validate JWT and attach user context — the BFF is the auth boundary.

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import { jwtVerify, createRemoteJWKSet } from 'jose';

const JWKS = createRemoteJWKSet(new URL(process.env.AUTH_JWKS_URL!));

export async function authenticate(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing token' });
  }

  try {
    const token = header.slice(7);
    const { payload } = await jwtVerify(token, JWKS, {
      issuer: process.env.AUTH_ISSUER,
      audience: process.env.AUTH_AUDIENCE,
    });

    req.user = {
      id: payload.sub!,
      email: payload.email as string,
      roles: (payload.roles ?? []) as string[],
    };

    next();
  } catch {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
}

// Role guard factory
export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.some(r => req.user?.roles.includes(r))) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}
```

---

### 8. Rate Limiting

Per-client and per-route rate limiting.

```typescript
// src/middleware/rate-limit.ts
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../cache/redis-cache';

export const globalLimit = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redis.sendCommand(args) }),
});

// Tighter limit for expensive aggregation endpoints
export const heavyLimit = rateLimit({
  windowMs: 60 * 1000,
  max: 20,
  keyGenerator: req => req.user?.id ?? req.ip, // per-user, not per-IP
});

// Apply in app.ts
app.use(globalLimit);
app.get('/api/dashboard', authenticate, heavyLimit, dashboardHandler);
```

---

### 9. Error Normalization

Translate downstream errors into consistent client-facing error shapes.

```typescript
// src/middleware/error-handler.ts
import { Request, Response, NextFunction } from 'express';
import { logger } from '../logger';

interface AppError extends Error {
  status?: number;
  code?: string;
}

export function errorHandler(
  err: AppError,
  req: Request,
  res: Response,
  _next: NextFunction
) {
  const status = err.status ?? 500;

  // Don't leak internal error details on 5xx
  const message = status < 500
    ? err.message
    : 'An unexpected error occurred';

  logger.error({ err, path: req.path, method: req.method }, 'Request failed');

  res.status(status).json({
    error: {
      message,
      code: err.code ?? 'INTERNAL_ERROR',
      requestId: req.id,  // useful for log correlation
    },
  });
}
```

---

### 10. GraphQL BFF

Use Apollo Server as a GraphQL BFF aggregating REST microservices.

```typescript
// src/graphql/server.ts
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { userClient, orderClient } from '../services/http-client';

const typeDefs = `#graphql
  type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order!]!
  }

  type Order {
    id: ID!
    total: Float!
    status: String!
    date: String!
  }

  type Query {
    me: User
    dashboard: DashboardData
  }

  type DashboardData {
    user: User!
    recentOrders: [Order!]!
    totalSpent: Float!
  }
`;

const resolvers = {
  Query: {
    me: async (_: unknown, __: unknown, ctx: any) => {
      const { data } = await userClient.get(`/users/${ctx.user.id}`);
      return data;
    },

    dashboard: async (_: unknown, __: unknown, ctx: any) => {
      const [userRes, ordersRes] = await Promise.all([
        userClient.get(`/users/${ctx.user.id}`),
        orderClient.get(`/orders?userId=${ctx.user.id}&limit=5`),
      ]);

      return {
        user: userRes.data,
        recentOrders: ordersRes.data.items,
        totalSpent: ordersRes.data.totalSpent,
      };
    },
  },

  User: {
    // Resolver-level data fetching — only called if client requests orders field
    orders: async (user: any) => {
      const { data } = await orderClient.get(`/orders?userId=${user.id}`);
      return data.items;
    },
  },
};

export const apolloServer = new ApolloServer({ typeDefs, resolvers });
```

---

## Best Practices

### Data & API Design
- **One BFF per client type** — resist the temptation to share a BFF between web and mobile; their needs will diverge
- **Shape responses for the screen, not the domain** — return exactly what the UI needs, nothing more
- **Use `Promise.allSettled` for non-critical data** — partial responses are better than total failures
- **Validate inputs with Zod or Joi** — the BFF is a system boundary; validate all incoming request shapes

### Performance
- **Fan out in parallel** (`Promise.all`) whenever downstream calls are independent
- **Cache aggressively at the BFF** — downstream services are slow and shared; caching at the BFF is safe and effective
- **Set timeouts on every downstream call** — never let a slow service hang your BFF process
- **Use HTTP/2 or connection pools** for internal service communication

### Reliability
- **Wrap downstream calls in circuit breakers** — use opossum or similar
- **Always provide fallback values** — return stale or default data rather than 5xx when non-critical services are down
- **Never propagate internal 500s raw** — normalize errors before sending to the client

### Security
- **The BFF is the auth boundary** — validate tokens here, never trust the client
- **Strip internal fields** in transformers — never accidentally expose `internal_ref`, `cost_price`, etc.
- **Use helmet** for HTTP security headers
- **Never log full request bodies** — they may contain PII or credentials

### Observability
- **Add a request ID** to every request (use `uuid` or `nanoid`) and thread it through downstream calls via headers
- **Use structured logging** (Pino, Winston JSON mode) — not `console.log`
- **Emit metrics** per route: latency, error rate, downstream call count
- **Trace with OpenTelemetry** — the BFF is the ideal place to start a trace span

```typescript
// Structured logging with Pino
import pino from 'pino';
export const logger = pino({ level: process.env.LOG_LEVEL ?? 'info' });

// Request ID middleware
import { randomUUID } from 'crypto';
app.use((req, _res, next) => {
  req.id = req.headers['x-request-id'] as string ?? randomUUID();
  next();
});
```

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| **Shared BFF across client types** | Each client type gets its own BFF — different needs will always emerge |
| **Business logic in BFF** | BFF transforms data; it doesn't own domain logic. Push business rules to downstream services |
| **No timeout on downstream calls** | Always set `timeout` in your HTTP client config |
| **Awaiting in a loop** | Use `Promise.all` / `Promise.allSettled` for independent calls |
| **Returning raw downstream errors to client** | Normalize all errors in an express error handler |
| **No caching** | BFF is the perfect caching layer — cache aggressively with short TTLs |
| **Fat BFF with its own database** | BFF should be stateless; if you need a DB, reconsider whether this is a real microservice |
| **Ignoring observability** | Without request IDs and structured logs, debugging fan-out failures is nearly impossible |
| **Over-fetching from downstream** | Only request fields you need — use query params, GraphQL fragments, or field masks |

---

## Folder Structure

```
src/
├── app.ts                  # Express app setup, middleware registration
├── server.ts               # HTTP server bootstrap
├── config.ts               # Env var validation (zod/dotenv)
│
├── routes/                 # One file per BFF feature/screen
│   ├── dashboard.route.ts
│   ├── profile.route.ts
│   └── orders.route.ts
│
├── services/               # HTTP clients for each downstream service
│   ├── http-client.ts      # Axios factory with interceptors
│   ├── user.service.ts
│   ├── order.service.ts
│   └── inventory.service.ts
│
├── transformers/           # Response shaping — raw domain → client shape
│   ├── user.transformer.ts
│   └── order.transformer.ts
│
├── middleware/
│   ├── auth.ts             # JWT validation, req.user attachment
│   ├── rate-limit.ts       # Rate limit configs
│   └── error-handler.ts    # Global error normalizer
│
├── cache/
│   └── redis-cache.ts      # Cache helpers (get/set/invalidate)
│
├── graphql/                # If using GraphQL BFF
│   ├── schema.ts
│   ├── resolvers/
│   └── server.ts
│
└── logger.ts               # Pino instance
```
