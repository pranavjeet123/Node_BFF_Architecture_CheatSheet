# Node-RED: A Practical BFF Approach

> Based on: *Node-RED: A Practical BFF Approach* by Swagata Acharyya (April 2025)

## Table of Contents
- [What is Node-RED?](#what-is-node-red)
- [The Wedding Planner Analogy](#the-wedding-planner-analogy)
- [Why Node-RED as a BFF?](#why-node-red-as-a-bff)
- [Node-RED Architecture](#node-red-architecture)
- [Flow-Based BFF Design](#flow-based-bff-design)
- [Core Node-RED BFF Patterns](#core-node-red-bff-patterns)
- [Real-World Use Cases](#real-world-use-cases)
- [Benefits & Tradeoffs](#benefits--tradeoffs)
- [Node-RED vs Custom Node.js BFF](#node-red-vs-custom-nodejs-bff)
- [Production Considerations](#production-considerations)

---

## What is Node-RED?

Node-RED is a **flow-based, low-code programming tool** originally built for IoT workflows. It provides a browser-based drag-and-drop editor where data pipelines are constructed visually as connected **nodes** — each node performs a discrete operation on a message.

```
[HTTP In] ──► [Function] ──► [HTTP Request] ──► [Change] ──► [HTTP Response]
              (shape req)    (call service)    (shape res)
```

It runs on **Node.js** with **Express.js** as its HTTP layer, which means it inherits the full I/O performance characteristics that make Node.js ideal for BFF workloads — and it can be extended with any npm package.

---

## The Wedding Planner Analogy

The author describes the BFF role with a memorable analogy:

> *"A BFF is like a wedding planner. The planner handles all the backend chaos — orchestration, data, business rules — so the frontend (the couple) can focus purely on the experience."*

| Wedding Planner | BFF Layer |
|---|---|
| Coordinates vendors (florist, caterer, band) | Orchestrates microservices (user, orders, inventory) |
| Delivers a single coherent event | Delivers a single aggregated API response |
| Hides supplier negotiation from the couple | Hides service complexity from the frontend |
| Manages timeline and dependencies | Manages call ordering and parallelism |
| Handles last-minute failures | Implements fallbacks and circuit breakers |

---

## Why Node-RED as a BFF?

The core problem it addresses: **mobile apps that need data from multiple services**.

A typical mobile dashboard screen might require:

```
┌─────────────────────────────┐
│  Mobile App Dashboard       │
│                             │
│  [User Profile]             │  ← User Service
│  [Wallet Balance]           │  ← Wallet Service
│  [Active Accounts]          │  ← Accounts Service
│  [Loan Summary]             │  ← Loans Service
│  [Deposit Details]          │  ← Deposits Service
└─────────────────────────────┘
```

Without a BFF, the mobile client makes **5 separate HTTP calls** across the internet, each adding latency, battery drain, and failure risk.

With a Node-RED BFF:

```
Mobile App  ──►  [Node-RED BFF]  ──►  User Service
                                  ──►  Wallet Service    (parallel)
                                  ──►  Accounts Service  (parallel)
                                  ──►  Loans Service     (parallel)
                                  ──►  Deposits Service  (parallel)
                      │
                      ▼
              Aggregated, shaped response
```

The mobile app makes **1 call**. Node-RED fans out to all 5 services in parallel and returns a single response shaped for the screen.

---

## Node-RED Architecture

### Core Building Blocks

```
┌──────────────────────────────────────────┐
│           Node-RED Runtime               │
│                                          │
│  ┌────────────┐   ┌──────────────────┐   │
│  │  Express   │   │   Flow Engine     │   │
│  │  HTTP Server│   │  (message router) │   │
│  └────────────┘   └──────────────────┘   │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │             Node Library           │  │
│  │  [HTTP In] [Function] [HTTP Req]   │  │
│  │  [Switch] [Change] [Join] [Split]  │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌───────────────┐  ┌─────────────────┐  │
│  │ Local Context │  │ Global Context  │  │
│  │ (per flow)    │  │ (across flows)  │  │
│  └───────────────┘  └─────────────────┘  │
└──────────────────────────────────────────┘
```

### The Message Object (`msg`)

Every node in a flow receives and passes a `msg` object. The convention:

```javascript
{
  payload: { ... },       // primary data being processed
  req: { ... },           // original HTTP request (on HTTP In nodes)
  res: { ... },           // HTTP response handle
  headers: { ... },       // HTTP headers
  statusCode: 200,
  // any custom properties you add
}
```

### Function Node

The Function node is where custom JavaScript logic lives:

```javascript
// Accessing and transforming the payload
msg.payload = msg.payload.toUpperCase();
return msg;

// Branching — return array to route to different outputs
if (msg.payload.type === 'premium') {
  return [msg, null];   // route to output 1
} else {
  return [null, msg];   // route to output 2
}

// Accessing context
const cache = flow.get('tokenCache') || {};
flow.set('tokenCache', cache);
```

---

## Flow-Based BFF Design

### A Complete Dashboard Flow

```
[HTTP In /dashboard]
         │
         ▼
[Function: extract userId from headers]
         │
    ┌────┴────────────────────┐
    │                         │
    ▼                         ▼
[HTTP Request: User Svc]   [HTTP Request: Wallet Svc]
    │                         │
    ▼                         ▼
[Change: reshape user]    [Change: reshape balance]
    │                         │
    └─────────┬───────────────┘
              │
              ▼
         [Join: wait for all]
              │
              ▼
    [Function: assemble response]
              │
              ▼
         [HTTP Response]
```

### Parallel Service Calls in Node-RED

Node-RED handles parallelism visually — splitting a flow into multiple branches that reconnect at a **Join** node.

```
                    ┌──► [User Service]    ──┐
                    │                       │
[Split / Fork] ─────┼──► [Wallet Service]  ──┼──► [Join] ──► [Assemble]
                    │                       │
                    └──► [Accounts Service] ──┘
```

The **Join** node collects all messages and emits a single combined message when all branches complete.

---

## Core Node-RED BFF Patterns

### Pattern 1: Token Injection

Intercept the client's session token and inject it into downstream service headers — the client never talks to services directly.

```javascript
// Function node: inject auth header into downstream request
const sessionToken = msg.req.headers['authorization'];

msg.headers = {
  'Authorization': sessionToken,
  'X-Internal-Token': env.get('INTERNAL_SERVICE_TOKEN'),
  'X-Request-Id': msg.req.headers['x-request-id'] || generateId(),
};

return msg;
```

### Pattern 2: Response Shaping

Transform raw domain data into a client-optimized shape.

```javascript
// Function node: shape user service response for mobile
const raw = msg.payload;

msg.payload = {
  id: raw.user_id,
  name: `${raw.first_name} ${raw.last_name}`,
  avatar: raw.profile_image_url,
  // Strip internal fields: raw.internal_ref, raw.audit_log, etc.
};

return msg;
```

### Pattern 3: Conditional Routing

Route requests to different downstream services based on request properties.

```javascript
// Switch node condition examples:
// msg.payload.accountType === 'premium'  → premium service
// msg.payload.accountType === 'standard' → standard service

// Or in a Function node:
if (msg.payload.region === 'IN') {
  msg.url = 'https://in.payments-service.internal/pay';
} else {
  msg.url = 'https://global.payments-service.internal/pay';
}
return msg;
```

### Pattern 4: Error Handling Flow

Dedicated error branch using Node-RED's Catch node.

```
[HTTP In] ──► [Function] ──► [HTTP Request] ──► [HTTP Response: 200]
                                    │
                              [Catch node]
                                    │
                            [Function: format error]
                                    │
                            [HTTP Response: 4xx/5xx]
```

```javascript
// Function node in error branch
msg.statusCode = msg.error?.statusCode || 500;
msg.payload = {
  error: {
    message: msg.statusCode < 500 ? msg.error.message : 'Unexpected error',
    code: msg.error?.code || 'INTERNAL_ERROR',
  },
};
return msg;
```

### Pattern 5: Caching with Flow Context

Use Node-RED's flow context as an in-memory cache (or use the Redis context store for persistence).

```javascript
// Function node: cache-aside pattern
const cacheKey = `profile_${msg.payload.userId}`;
const cached = flow.get(cacheKey);

if (cached && cached.expiresAt > Date.now()) {
  msg.payload = cached.data;
  msg.fromCache = true;
  return [msg, null];  // output 1: cache hit, skip HTTP call
}

// output 2: cache miss, proceed to HTTP request
return [null, msg];
```

```javascript
// Function node: after HTTP Request, store in cache
flow.set(`profile_${msg.userId}`, {
  data: msg.payload,
  expiresAt: Date.now() + 5 * 60 * 1000,  // 5 min TTL
});
return msg;
```

---

## Real-World Use Cases

### 1. Mobile Dashboard Aggregation
Combine user profile, balance, accounts, loans, and deposits into one payload shaped for the home screen. Parallel branches + Join ensures minimum latency.

### 2. Session Token Management
The BFF intercepts the client's OAuth token, validates it, and injects it into every downstream call as a header — downstream services receive pre-authenticated requests without the mobile app managing service-specific auth.

### 3. Server-Driven UI (SDUI)
The BFF returns not just data but **layout instructions** — which components to render, in what order, with what data. Node-RED flows can easily build these composite payloads.

```javascript
// Server-Driven UI response shape
msg.payload = {
  screen: 'home',
  components: [
    { type: 'ProfileHeader', data: user },
    { type: 'BalanceCard', data: wallet },
    { type: 'AccountList', data: accounts.slice(0, 3) },
    { type: 'PromoBanner', data: offers[0] },
  ],
};
```

### 4. A/B Test Routing
Evaluate feature flags in the BFF and route to different service versions or response shapes without touching the mobile client.

```javascript
const variant = getABVariant(msg.payload.userId);  // 'control' | 'test'
msg.url = variant === 'test'
  ? env.get('NEW_CHECKOUT_SERVICE_URL')
  : env.get('CHECKOUT_SERVICE_URL');
return msg;
```

---

## Benefits & Tradeoffs

### Benefits

| Benefit | Detail |
|---|---|
| **Visual accessibility** | Non-engineers (product, QA) can read and understand flows |
| **Rapid prototyping** | A new aggregation endpoint can be wired in minutes without code scaffolding |
| **Modular flows** | Flows are composable — share sub-flows across endpoints |
| **Built-in nodes** | HTTP, JSON, CSV, MQTT, WebSocket, debug nodes ship out of the box |
| **npm ecosystem** | Install any npm package and use it in Function nodes |
| **Low ceremony** | No boilerplate — drop an HTTP In node and you have an endpoint |

### Tradeoffs

| Challenge | Mitigation |
|---|---|
| **High-load scalability** | Run Node-RED behind a reverse proxy (nginx); use PM2 cluster mode or Kubernetes HPA |
| **Code review friction** | Flows are JSON under the hood — export to `.json` and diff in git |
| **Version control** | Commit `flows.json` to git; use Node-RED's Projects feature for proper git integration |
| **Complex logic** | Heavy business logic belongs in downstream services, not Function nodes |
| **Observability** | Use the `node-red-contrib-opentelemetry` node or instrument via custom Function nodes |
| **Testing** | Use `node-red-contrib-testing` or extract logic to external modules with unit tests |

---

## Node-RED vs Custom Node.js BFF

| Dimension | Node-RED | Custom Node.js (Express/Fastify) |
|---|---|---|
| **Setup speed** | Minutes | Hours |
| **Prototype → prod** | Very fast | Moderate |
| **Complex logic** | Awkward in Function nodes | Native |
| **Team accessibility** | High (visual) | Developer-only |
| **Testability** | Limited | Full unit/integration testing |
| **Performance tuning** | Constrained | Full control |
| **CI/CD maturity** | Growing | Mature |
| **Best for** | Rapid aggregation, IoT, SDUI | High-throughput, complex transformation |

**Rule of thumb:** Use Node-RED when speed of delivery and team accessibility matter most. Use a custom Node.js BFF when you need full test coverage, complex business logic, or strict performance SLAs.

---

## Production Considerations

### Running Node-RED at Scale

```bash
# PM2 cluster mode — utilize all CPU cores
pm2 start node-red --name bff -- -p 1880
pm2 scale bff max

# Or with a custom settings file
pm2 start node-red --name bff -- --settings /app/settings.js
```

### settings.js Essentials

```javascript
module.exports = {
  uiPort: process.env.PORT || 1880,
  uiHost: '0.0.0.0',

  // Disable editor in production
  disableEditor: process.env.NODE_ENV === 'production',

  // Use Redis for context storage (shared across cluster nodes)
  contextStorage: {
    default: { module: 'memory' },
    redis: {
      module: require('node-red-context-redis'),
      config: { host: process.env.REDIS_HOST, port: 6379 },
    },
  },

  // Secure the admin API
  adminAuth: {
    type: 'credentials',
    users: [{ username: 'admin', password: process.env.ADMIN_PWD_HASH, permissions: '*' }],
  },

  logging: {
    console: { level: 'info', metric: false, audit: false },
  },
};
```

### Version-Controlling Flows

```bash
# Export flows via CLI
node-red-admin exportFlows --host localhost --port 1880 > flows.json

# Or enable Node-RED Projects (built-in git integration)
# settings.js:
editorTheme: {
  projects: { enabled: true }
}
```
