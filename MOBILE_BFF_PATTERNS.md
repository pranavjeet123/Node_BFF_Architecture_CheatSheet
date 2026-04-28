# Mobile BFF Patterns — Aggregation, SDUI & Token Management

> Patterns extracted and expanded from the Node-RED BFF approach by Swagata Acharyya.
> Applicable to any Node.js BFF (Node-RED, Express, Fastify, NestJS).

## Table of Contents
- [The Mobile BFF Problem](#the-mobile-bff-problem)
- [Pattern 1: Dashboard Aggregation](#pattern-1-dashboard-aggregation)
- [Pattern 2: Session Token Passthrough](#pattern-2-session-token-passthrough)
- [Pattern 3: Server-Driven UI (SDUI)](#pattern-3-server-driven-ui-sdui)
- [Pattern 4: Partial Success / Graceful Degradation](#pattern-4-partial-success--graceful-degradation)
- [Pattern 5: Mobile-Specific Response Shaping](#pattern-5-mobile-specific-response-shaping)
- [Pattern 6: Offline-Friendly Caching](#pattern-6-offline-friendly-caching)
- [Pattern 7: A/B Testing & Feature Flags in the BFF](#pattern-7-ab-testing--feature-flags-in-the-bff)
- [Pattern 8: Pagination Translation](#pattern-8-pagination-translation)
- [Mobile vs Web BFF — Key Differences](#mobile-vs-web-bff--key-differences)

---

## The Mobile BFF Problem

A mobile dashboard screen needs data from 5 separate services:

```
Home Screen Requirements
├── User Profile        ← User Service
├── Wallet Balance      ← Wallet Service
├── Active Accounts     ← Accounts Service
├── Loan Summary        ← Loans Service
└── Latest Offers       ← Promotions Service
```

### Without BFF — 5 serial/parallel client-side calls

```
Mobile App (on 4G, 80ms RTT)

GET /user/profile         → 80ms + 120ms service = 200ms
GET /wallet/balance       → 80ms + 95ms service  = 175ms   (if parallel)
GET /accounts             → 80ms + 140ms service = 220ms   (if parallel)
GET /loans/summary        → 80ms + 200ms service = 280ms   (if parallel)
GET /promotions/offers    → 80ms + 60ms service  = 140ms   (if parallel)

Total (parallel):  280ms + 80ms overhead  =  ~360ms
Total (serial):    200+175+220+280+140    = ~1015ms
Battery impact:    5 radios activations
Code complexity:   client manages error state for 5 calls
```

### With BFF — 1 client call, 5 internal parallel calls

```
Mobile App (on 4G, 80ms RTT)

GET /api/home  → 80ms client-to-BFF + 280ms (BFF fans out, takes slowest) = ~360ms

But:
- 1 radio activation instead of 5
- Client handles 1 error, not 5
- BFF can return partial data if 1 service fails
- Payload pre-shaped for the screen
```

---

## Pattern 1: Dashboard Aggregation

Aggregate multiple service calls into one endpoint per screen.

```typescript
// src/routes/home.route.ts
import { Router } from 'express';
import { userService, walletService, accountsService, loansService, promoService } from '../services';

const router = Router();

router.get('/home', async (req, res, next) => {
  const userId = req.user.id;
  const deviceType = req.headers['x-device-type'] as string;  // 'ios' | 'android'

  try {
    // Fan out — all calls in parallel
    const [user, wallet, accounts, loans, promos] = await Promise.allSettled([
      userService.getProfile(userId),
      walletService.getBalance(userId),
      accountsService.list(userId),
      loansService.getSummary(userId),
      promoService.getOffers(userId, { deviceType }),
    ]);

    res.json({
      user:     fulfilled(user,     null),
      wallet:   fulfilled(wallet,   { balance: null, currency: 'INR' }),
      accounts: fulfilled(accounts, []),
      loans:    fulfilled(loans,    null),
      offers:   fulfilled(promos,   []),
    });
  } catch (err) {
    next(err);
  }
});

// Helper: extract value from allSettled result, return fallback on rejection
function fulfilled<T>(result: PromiseSettledResult<T>, fallback: T): T {
  return result.status === 'fulfilled' ? result.value : fallback;
}

export default router;
```

---

## Pattern 2: Session Token Passthrough

The BFF validates the client token and injects service-specific auth into downstream requests. The mobile app never manages inter-service credentials.

```typescript
// src/middleware/auth.ts — validate incoming token
import { jwtVerify, createRemoteJWKSet } from 'jose';

const JWKS = createRemoteJWKSet(new URL(process.env.AUTH_JWKS_URL!));

export async function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Missing token' });

  try {
    const { payload } = await jwtVerify(token, JWKS);
    req.user = { id: payload.sub, email: payload.email };
    req.rawToken = token;  // keep raw token for passthrough
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

```typescript
// src/services/http-client.ts — inject auth into every downstream call
import axios from 'axios';

export function createServiceClient(baseURL: string) {
  const client = axios.create({ baseURL, timeout: 5000 });

  client.interceptors.request.use(config => {
    // Internal service-to-service auth (separate from client token)
    config.headers['X-Internal-Token'] = process.env.INTERNAL_SERVICE_TOKEN!;

    // Pass-through the original user token for downstream audit logging
    if (config.headers['X-User-Token']) {
      // already set by caller — respect it
    }

    // Correlation ID for distributed tracing
    config.headers['X-Request-Id'] = config.headers['X-Request-Id'] ?? generateId();

    return config;
  });

  return client;
}

// In route handlers — forward user context to services
const userData = await userClient.get(`/users/${userId}`, {
  headers: { 'X-User-Token': req.rawToken },
});
```

---

## Pattern 3: Server-Driven UI (SDUI)

Instead of returning raw data, the BFF returns a **layout specification** — which UI components to render and what data to populate them with. The mobile client becomes a renderer, not a decision-maker.

```typescript
// src/routes/sdui-home.route.ts
router.get('/screen/home', async (req, res, next) => {
  const userId = req.user.id;
  const appVersion = req.headers['x-app-version'] as string;
  const platform = req.headers['x-platform'] as 'ios' | 'android';

  try {
    const [user, wallet, accounts, offers] = await Promise.all([
      userService.getProfile(userId),
      walletService.getBalance(userId),
      accountsService.list(userId),
      promoService.getOffers(userId),
    ]);

    const components = [];

    // Header is always shown
    components.push({
      type: 'ProfileHeader',
      id: 'profile-header',
      data: { name: user.firstName, avatar: user.avatarUrl, tier: user.tier },
    });

    // Wallet shown only if user has an active wallet
    if (wallet.status === 'active') {
      components.push({
        type: 'WalletCard',
        id: 'wallet-card',
        data: { balance: wallet.balance, currency: wallet.currency },
        action: { type: 'navigate', route: '/wallet' },
      });
    }

    // Account list — cap at 3 items on mobile
    if (accounts.length > 0) {
      components.push({
        type: 'AccountList',
        id: 'account-list',
        data: {
          items: accounts.slice(0, 3).map(a => ({
            id: a.id,
            type: a.accountType,
            last4: a.accountNumber.slice(-4),
            balance: a.balance,
          })),
          showViewAll: accounts.length > 3,
        },
      });
    }

    // Promotions banner — A/B test: only show on new app versions
    if (offers.length > 0 && semver.gte(appVersion, '3.2.0')) {
      components.push({
        type: 'PromoBanner',
        id: 'promo-banner',
        data: offers[0],
        analytics: { event: 'promo_impression', offerId: offers[0].id },
      });
    }

    res.json({
      screen: 'home',
      version: 1,
      components,
      metadata: { refreshAfterSeconds: 300 },
    });
  } catch (err) {
    next(err);
  }
});
```

**Client-side rendering (React Native):**
```typescript
// Renderer is purely declarative — no business logic on device
const componentMap = {
  ProfileHeader: ProfileHeaderComponent,
  WalletCard: WalletCardComponent,
  AccountList: AccountListComponent,
  PromoBanner: PromoBannerComponent,
};

function HomeScreen({ components }) {
  return (
    <ScrollView>
      {components.map(({ type, id, data, action }) => {
        const Component = componentMap[type];
        return <Component key={id} data={data} action={action} />;
      })}
    </ScrollView>
  );
}
```

---

## Pattern 4: Partial Success / Graceful Degradation

Never fail the whole response because one non-critical service is down.

```typescript
router.get('/home', async (req, res, next) => {
  const userId = req.user.id;

  const [userRes, walletRes, offersRes] = await Promise.allSettled([
    userService.getProfile(userId),    // critical
    walletService.getBalance(userId),  // important
    promoService.getOffers(userId),    // non-critical
  ]);

  // If a critical service fails, return an error
  if (userRes.status === 'rejected') {
    return next(userRes.reason);
  }

  const response: any = {
    user: userRes.value,
  };

  // Degrade gracefully for non-critical services
  if (walletRes.status === 'fulfilled') {
    response.wallet = walletRes.value;
  } else {
    response.wallet = null;
    response._warnings = response._warnings ?? [];
    response._warnings.push({ service: 'wallet', message: 'Balance temporarily unavailable' });
  }

  response.offers = offersRes.status === 'fulfilled' ? offersRes.value : [];

  res.json(response);
});
```

---

## Pattern 5: Mobile-Specific Response Shaping

Mobile has different constraints than web — smaller screen, less memory, bandwidth cost.

```typescript
// src/transformers/account.transformer.ts

interface RawAccount {
  account_uuid: string;
  account_number: string;          // never send full to mobile
  account_type_code: string;
  current_balance_cents: number;
  available_balance_cents: number;
  interest_rate_bps: number;       // basis points — convert for display
  internal_risk_flag: string;      // NEVER send to client
  audit_created_by: string;        // NEVER send to client
  last_transaction_timestamp: string;
}

// Web BFF shape — rich, more fields
export function toWebAccount(raw: RawAccount) {
  return {
    id: raw.account_uuid,
    accountNumber: raw.account_number,      // web shows full number
    type: mapAccountType(raw.account_type_code),
    currentBalance: raw.current_balance_cents / 100,
    availableBalance: raw.available_balance_cents / 100,
    interestRate: raw.interest_rate_bps / 100,
    lastActivity: raw.last_transaction_timestamp,
  };
}

// Mobile BFF shape — minimal, display-ready
export function toMobileAccount(raw: RawAccount) {
  return {
    id: raw.account_uuid,
    last4: raw.account_number.slice(-4),    // mobile shows masked number
    type: mapAccountType(raw.account_type_code),
    balance: raw.available_balance_cents / 100,   // only available balance
    displayBalance: formatCurrency(raw.available_balance_cents / 100),
    // interest_rate, full account_number, timestamps omitted
  };
}

function mapAccountType(code: string): string {
  const types: Record<string, string> = {
    'SAV': 'Savings',
    'CUR': 'Current',
    'FD': 'Fixed Deposit',
    'RD': 'Recurring Deposit',
  };
  return types[code] ?? code;
}

function formatCurrency(amount: number): string {
  return new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' }).format(amount);
}
```

---

## Pattern 6: Offline-Friendly Caching

Mobile clients go offline. The BFF can add `Cache-Control` and `ETag` headers so the native HTTP layer (or a CDN) serves stale data when the network is down.

```typescript
import { createHash } from 'crypto';

router.get('/profile/:id', authenticate, async (req, res, next) => {
  try {
    const profile = await userService.getProfile(req.params.id);
    const body = JSON.stringify(profile);

    // ETag for conditional requests (If-None-Match)
    const etag = `"${createHash('md5').update(body).digest('hex')}"`;

    if (req.headers['if-none-match'] === etag) {
      return res.status(304).send();  // client's cache is still fresh
    }

    res
      .set('ETag', etag)
      .set('Cache-Control', 'private, max-age=300, stale-while-revalidate=600')
      .set('Last-Modified', new Date().toUTCString())
      .json(profile);
  } catch (err) {
    next(err);
  }
});
```

**BFF-level Redis cache with stale-while-revalidate:**
```typescript
export async function cachedWithSWR<T>(
  key: string,
  freshTTL: number,
  staleTTL: number,
  fn: () => Promise<T>
): Promise<{ data: T; stale: boolean }> {
  const raw = await redis.get(key);

  if (raw) {
    const { data, cachedAt } = JSON.parse(raw);
    const age = (Date.now() - cachedAt) / 1000;

    if (age < freshTTL) {
      return { data, stale: false };
    }

    if (age < staleTTL) {
      // Return stale immediately, refresh in background
      fn().then(fresh => redis.setEx(key, staleTTL, JSON.stringify({ data: fresh, cachedAt: Date.now() })));
      return { data, stale: true };
    }
  }

  const fresh = await fn();
  await redis.setEx(key, staleTTL, JSON.stringify({ data: fresh, cachedAt: Date.now() }));
  return { data: fresh, stale: false };
}
```

---

## Pattern 7: A/B Testing & Feature Flags in the BFF

Evaluate experiments in the BFF — not on-device — so you can change behavior without an app store release.

```typescript
// src/services/feature-flag.service.ts
import { getFlag } from './launchdarkly-client'; // or your FF provider

export async function evaluateFlags(userId: string, deviceInfo: DeviceInfo) {
  const context = {
    kind: 'user',
    key: userId,
    appVersion: deviceInfo.appVersion,
    platform: deviceInfo.platform,
    country: deviceInfo.country,
  };

  const [newCheckoutEnabled, promoV2Enabled] = await Promise.all([
    getFlag('new-checkout-flow', context, false),
    getFlag('promo-banner-v2', context, false),
  ]);

  return { newCheckoutEnabled, promoV2Enabled };
}
```

```typescript
// In route handler
router.get('/home', authenticate, async (req, res, next) => {
  const userId = req.user.id;
  const deviceInfo = extractDeviceInfo(req.headers);

  const [homeData, flags] = await Promise.all([
    aggregateHomeData(userId),
    evaluateFlags(userId, deviceInfo),
  ]);

  res.json({
    ...homeData,
    features: {
      newCheckout: flags.newCheckoutEnabled,
      promoV2: flags.promoV2Enabled,
    },
  });
});
```

---

## Pattern 8: Pagination Translation

Microservices use different pagination styles (cursor, offset, page). Normalize them for the mobile client.

```typescript
// Translate cursor-based (service) ↔ page-based (mobile client)

router.get('/transactions', authenticate, async (req, res, next) => {
  const page = parseInt(req.query.page as string) || 1;
  const pageSize = parseInt(req.query.pageSize as string) || 20;

  try {
    // Service uses cursor — translate from page number
    const cursor = pageNumberToCursor(page, pageSize);  // your encoding strategy

    const result = await transactionService.list({
      userId: req.user.id,
      cursor,
      limit: pageSize,
    });

    res.json({
      items: result.transactions.map(toMobileTransaction),
      pagination: {
        page,
        pageSize,
        total: result.totalCount,
        totalPages: Math.ceil(result.totalCount / pageSize),
        hasNext: result.hasNextPage,
        hasPrev: page > 1,
      },
    });
  } catch (err) {
    next(err);
  }
});
```

---

## Mobile vs Web BFF — Key Differences

| Concern | Mobile BFF | Web BFF |
|---|---|---|
| **Payload size** | Minimize aggressively — bandwidth and battery cost | Richer payloads are acceptable |
| **Auth mechanism** | JWT in Authorization header, no cookies | Cookies + CSRF tokens (web sessions) |
| **Caching strategy** | HTTP cache headers for native layer + offline support | Server-side cache, CDN |
| **Response shaping** | Display-ready strings (formatted currency, dates) | Raw values — formatting is browser's job |
| **Image fields** | Include CDN thumbnail URLs sized for mobile | Include original / hi-res URLs |
| **Error format** | Include user-facing message string for toast | Include code for i18n lookup |
| **Layout control** | Server-Driven UI (SDUI) | Frontend owns layout |
| **Screen data** | One endpoint per screen | One endpoint per resource (or GraphQL) |
| **Versioning** | Critical — old app versions must keep working | Less critical — browser updates automatically |
| **Pagination** | Cursor or infinite scroll | Page-based is fine |

### Versioning Strategy for Mobile BFF

Old app versions in the field can't be forced to update immediately. Version your BFF responses:

```typescript
router.get('/home', authenticate, (req, res, next) => {
  const appVersion = req.headers['x-app-version'] as string;

  if (semver.lt(appVersion, '3.0.0')) {
    return handleHomeV1(req, res, next);  // legacy shape
  }
  return handleHomeV2(req, res, next);  // current shape
});
```

Or use URL versioning:
```
GET /v1/home   → legacy mobile clients
GET /v2/home   → current clients
```

Deprecate old versions with a `Sunset` header:
```typescript
res.set('Sunset', 'Sat, 31 Dec 2025 23:59:59 GMT');
res.set('Deprecation', 'true');
```
