# Constants File Naming Conventions

## Filename Patterns (pick one and be consistent per project)

| Pattern          | Example             | When to use                                 |
| ---------------- | ------------------- | ------------------------------------------- |
| `constants.ts`   | `auth/constants.ts` | Co-located, single-topic module             |
| `*.constants.ts` | `auth.constants.ts` | When multiple `*.ts` files live in same dir |
| `constants/` dir | `auth/constants/`   | Multi-topic or large constant set           |
| `config.ts`      | `api/config.ts`     | Env-backed values, feature flags            |
| `enums.ts`       | `orders/enums.ts`   | Closed string/number domain sets only       |

Never mix patterns in the same project. Pick one and codify it in a comment at the top of
the first constants file created.

---

## Next.js (App Router)

```
src/
├── app/
│   └── (features)/
│       └── checkout/
│           ├── constants.ts          ← feature-scoped
│           └── ...
├── components/
│   └── ui/
│       └── constants.ts              ← UI token constants (if not using a theme system)
├── lib/
│   └── constants/
│       ├── index.ts                  ← barrel
│       ├── api.ts                    ← endpoints, timeouts
│       ├── env.ts                    ← all process.env reads
│       └── routes.ts                 ← typed route paths
└── types/
```

`lib/constants/env.ts` is the single place `process.env` is read:

```ts
// lib/constants/env.ts
export const ENV = {
  API_BASE_URL: process.env.NEXT_PUBLIC_API_URL!,
  STRIPE_KEY: process.env.NEXT_PUBLIC_STRIPE_KEY!,
  IS_PRODUCTION: process.env.NODE_ENV === "production",
  ENABLE_ANALYTICS: process.env.NEXT_PUBLIC_ENABLE_ANALYTICS === "true",
} as const;
```

---

## Expo / React Native

```
src/
├── features/
│   └── auth/
│       ├── constants.ts              ← AUTH_TOKEN_KEY, SESSION_TTL_MS, etc.
│       └── ...
├── navigation/
│   └── constants.ts                  ← SCREEN_NAMES, DEEP_LINK_PREFIXES
├── constants/
│   ├── index.ts
│   ├── api.ts
│   ├── storage.ts                    ← AsyncStorage keys
│   └── ui.ts                         ← spacing, font sizes (if not using NativeWind)
└── config/
    └── env.ts                        ← all expo-constants / process.env reads
```

---

## Plain Node.js / Express

```
src/
├── routes/
│   └── users/
│       ├── constants.ts              ← USER_ROLES, MAX_USERNAME_LENGTH
│       └── ...
├── middleware/
│   └── constants.ts                  ← RATE_LIMIT_WINDOW_MS, MAX_REQUESTS
├── constants/
│   ├── index.ts
│   ├── http.ts                       ← HTTP_STATUS, HTTP_METHODS
│   ├── errors.ts                     ← ERROR_CODES, ERROR_MESSAGES
│   └── db.ts                         ← QUERY_TIMEOUT_MS, MAX_CONNECTIONS
└── config/
    └── index.ts                      ← all process.env reads
```

---

## Monorepo (Turborepo / Nx)

```
packages/
├── constants/                        ← shared package, imported as @repo/constants
│   ├── package.json
│   ├── src/
│   │   ├── index.ts
│   │   ├── api.ts
│   │   ├── errors.ts
│   │   └── permissions.ts
│   └── tsconfig.json
├── web/
│   └── src/
│       └── features/
│           └── checkout/
│               └── constants.ts      ← app-specific, NOT in shared package
└── mobile/
    └── src/
        └── features/
            └── checkout/
                └── constants.ts
```

Shared constants (used by 2+ apps) → `packages/constants`.  
App-specific constants (used by 1 app only) → co-located with the feature.

---

## Barrel File Template

```ts
// constants/index.ts — generated barrel
// Re-export all groups so consumers can use either:
//   import { MAX_RETRY_COUNT } from '@/constants'          (convenient)
//   import { MAX_RETRY_COUNT } from '@/constants/api'      (tree-shakeable)

export * from "./api";
export * from "./errors";
export * from "./http";
export * from "./permissions";
export * from "./ui";
```

---

## Sorting / Ordering Within a File

Recommended order within a constants file:

1. Env-backed values first (they document external dependencies upfront)
2. Domain enums / `as const` objects
3. Numeric limits and thresholds
4. String keys (storage, query params, headers)
5. Derived/computed constants last (ones that reference earlier constants)

```ts
// ─── Env ──────────────────────────────────────────────────────────────────────
export const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? '';

// ─── Domain ───────────────────────────────────────────────────────────────────
export const ORDER_STATUS = { ... } as const;
export type OrderStatus = typeof ORDER_STATUS[keyof typeof ORDER_STATUS];

// ─── Limits ───────────────────────────────────────────────────────────────────
export const MAX_ORDER_ITEMS  = 50;
export const ORDER_TTL_MS     = 15 * 60 * 1_000; // 15 minutes

// ─── Keys ─────────────────────────────────────────────────────────────────────
export const PENDING_ORDER_KEY = 'pending_order_id';

// ─── Derived ──────────────────────────────────────────────────────────────────
export const OPEN_ORDER_STATUSES = [
  ORDER_STATUS.PENDING_PAYMENT,
  ORDER_STATUS.CONFIRMED,
] as const satisfies readonly OrderStatus[];
```
