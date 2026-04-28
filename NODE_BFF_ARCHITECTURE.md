# Node.js BFF (Backend for Frontend) Architecture

## Table of Contents
- [What is BFF?](#what-is-bff)
- [Why BFF Exists](#why-bff-exists)
- [Significance of BFF](#significance-of-bff)
- [When to Use BFF](#when-to-use-bff)
- [When NOT to Use BFF](#when-not-to-use-bff)
- [BFF vs API Gateway](#bff-vs-api-gateway)
- [Architecture Overview](#architecture-overview)
- [Node.js as the BFF Layer](#nodejs-as-the-bff-layer)

---

## What is BFF?

**Backend for Frontend (BFF)** is an architectural pattern where a dedicated backend service is created specifically for each frontend client (web, mobile, TV, etc.).

Instead of a single general-purpose API serving all clients, each frontend gets its own backend — shaped exactly to its needs.

```
Without BFF                        With BFF

[Web App]  ──────┐                 [Web App]  ──── [Web BFF]  ──┐
[iOS App]  ──────┼──► [API]        [iOS App]  ──── [Mobile BFF] ─┼──► [Microservices]
[Android]  ──────┘                 [Android]  ──── [Mobile BFF] ─┘
```

The term was coined by **Sam Newman** (author of *Building Microservices*) and is widely adopted in organizations running multiple client types against microservices backends.

---

## Why BFF Exists

### The Core Problem
A single shared API must satisfy the requirements of every client simultaneously. This creates friction:

- **Mobile clients** need lightweight payloads, battery-conscious polling strategies, and offline-sync support
- **Web clients** need rich data shapes, server-side rendering hints, and feature-flag responses
- **Third-party clients** need stable versioned contracts

Trying to serve all of these from one API leads to:

| Problem | Impact |
|---|---|
| Over-fetching | Mobile gets fields it never uses, wastes bandwidth |
| Under-fetching | Web makes 5 calls to assemble one screen |
| Versioning hell | A mobile breaking change forces a web deploy |
| Shared release cycles | Teams block each other |
| Generic error shapes | Clients parse errors differently anyway |

### The BFF Solution
Each client team owns its BFF. The BFF:
1. Aggregates calls to downstream microservices
2. Transforms and shapes data for that specific client
3. Handles authentication and session logic per client type
4. Decouples frontend release cycles from backend release cycles

---

## Significance of BFF

### 1. Client-Optimized Data Contracts
The BFF owns the response shape. No more "include everything and let the client filter."

```
Downstream microservices return raw domain data.
BFF composes, filters, and reshapes it for the exact screen that asked.
```

### 2. Team Autonomy
Frontend teams can evolve their BFF without coordinating with shared-API teams. The contract between BFF and the frontend is internal to the team.

### 3. Security Boundary
The BFF is the only service the frontend talks to. It:
- Validates tokens / sessions
- Strips sensitive fields before forwarding to the client
- Enforces rate limits per client type
- Implements CORS policy

### 4. Protocol Translation
Microservices might speak gRPC, GraphQL, REST, or message queues. The BFF translates everything into whatever the frontend needs (usually REST or GraphQL).

```
[React App] ──REST──► [Web BFF]
                          │
                ┌─────────┼─────────┐
              gRPC      REST      AMQP
                │         │         │
           [User Svc] [Order Svc] [Notif Svc]
```

### 5. Fault Isolation
A buggy downstream service affects only the BFF that calls it, not every client type. You can implement fallback/default data in the BFF independently.

### 6. Performance Gains
The BFF runs in the same datacenter as your microservices. Network hops between BFF and services are fast (sub-ms). The client makes one call; the BFF fans out internally.

```
Without BFF: Client makes 5 serial HTTP calls across the internet
With BFF:    Client makes 1 HTTP call; BFF makes 5 parallel internal calls
```

### 7. Experimentation & Feature Flags
The BFF is the right place to evaluate feature flags and A/B experiments before they reach the client, keeping that logic off the device.

---

## When to Use BFF

- You have **multiple distinct client types** (web, native mobile, smart TV, etc.)
- Your domain is split into **microservices** and clients need aggregated views
- Frontend teams want to **deploy independently** of backend teams
- You need **different auth/session strategies** per client (cookie-based for web, JWT for mobile)
- You want to **protect downstream services** from direct client access
- Your clients have **different performance budgets** (3G mobile vs broadband desktop)

---

## When NOT to Use BFF

- You have a **single client type** — a standard REST or GraphQL API is simpler
- Your application is a **simple CRUD app** with no aggregation need
- Your team is **small** and the overhead of maintaining multiple BFFs outweighs the benefit
- You are **not using microservices** — BFF shines when aggregating multiple services

---

## BFF vs API Gateway

These are often confused. They solve different problems.

| Concern | API Gateway | BFF |
|---|---|---|
| Purpose | Cross-cutting infrastructure (routing, rate limiting, auth) | Client-specific logic and aggregation |
| Owned by | Platform/infra team | Frontend team |
| Count | One (or a few) | One per client type |
| Business logic | None | Yes — shaping, aggregation, transformation |
| Protocol translation | Basic | Yes |

They are **complementary**, not alternatives. A common setup:

```
[Client] ──► [API Gateway] ──► [BFF] ──► [Microservices]
               (rate limit,        (aggregate,
                TLS, routing)       transform, auth session)
```

---

## Architecture Overview

### Single BFF (one client type)
```
[React SPA]
     │
     ▼
[Node.js Web BFF]
  - /api/dashboard  → aggregates User + Orders + Analytics
  - /api/profile    → User service + preferences
  - /api/cart       → Cart + Inventory + Pricing
     │
     ├──► [User Service]
     ├──► [Order Service]
     ├──► [Inventory Service]
     └──► [Pricing Service]
```

### Multiple BFFs (multiple client types)
```
[React Web]   ──► [Web BFF    :3001]  ──┐
[iOS App]     ──► [Mobile BFF :3002]  ──┼──► [Microservices]
[Partner API] ──► [Partner BFF:3003]  ──┘
```

---

## Node.js as the BFF Layer

Node.js is the dominant choice for BFF layers for several reasons:

| Reason | Detail |
|---|---|
| **Non-blocking I/O** | BFF workload is almost entirely I/O: HTTP calls, auth checks, caching. Node's event loop excels here. |
| **JSON native** | No serialization overhead — Node speaks JSON natively, matching REST/GraphQL payloads |
| **Same language as frontend** | TypeScript/JavaScript shared across client and BFF; teams can cross-contribute |
| **Rich ecosystem** | Express, Fastify, NestJS, Apollo Server — mature libraries for every BFF pattern |
| **Lightweight** | BFF containers are small, start fast, scale horizontally |
| **GraphQL tooling** | Apollo Server, graphql-yoga — best-in-class GraphQL BFF frameworks are Node-first |

### Recommended Stack

| Layer | Recommended Options |
|---|---|
| Framework | Fastify (performance) / NestJS (structure) / Express (simplicity) |
| HTTP Client | Axios / undici / got |
| GraphQL | Apollo Server / graphql-yoga |
| Caching | node-cache / ioredis |
| Auth | passport.js / jose (JWT) |
| Validation | Zod / Joi |
| Observability | OpenTelemetry + Pino logger |
