---
name: no-magic-values
description: >
  Eliminate magic values in TypeScript/JavaScript codebases by extracting literals into
  well-placed, well-named constants. Use this skill whenever the user asks to "remove magic
  values / numbers / strings", "extract constants", "clean up hardcoded values", or when
  writing or reviewing TypeScript/JavaScript code that contains inline literals that should
  be named. Also trigger when the user shares code with unexplained numbers, repeated string
  literals, hardcoded URLs or status codes, inline arrays that act as fixed sets, or env
  values that are hardcoded. This skill always scans for an existing constants file or
  directory before creating anything new — placement is as important as extraction. When a
  runtime type-checking library (Zod, Valibot, Arktype, Typebox) is present, always derive
  TypeScript types from the schema itself — never write a parallel hand-authored type.
---

# No Magic Values — TypeScript/JavaScript

The rule: **a literal that requires context to understand, or appears more than once, is a
magic value and must be named.** Extraction is only half the job — the other half is placing
the constant where future readers will find it naturally.

---

## 1. What Counts as a Magic Value

### Numbers
```ts
// ❌ Magic
setTimeout(fn, 3000);
if (retries > 5) throw new Error('...');
const chunk = data.slice(0, 100);
if (user.role === 2) { ... }
```

### Strings
```ts
// ❌ Magic
fetch('https://api.example.com/v2/users');
if (status === 'pending') { ... }
localStorage.setItem('auth_token', token);
throw new Error('RATE_LIMITED');
```

### Repeated literals (even once is enough if the meaning isn't obvious)
```ts
// ❌ — what does 86400 mean?
const ttl = 86400;
const expiry = now + 86400;
```

### Constant arrays (fixed sets that define a domain)
```ts
// ❌ Magic — this is a domain enum in disguise
const allowed = ['admin', 'editor', 'viewer'];
const METHODS = ['GET', 'POST', 'PUT', 'DELETE'];
```

### Hardcoded env values (should be in `.env`, referenced via a config constant)
```ts
// ❌ Magic — will differ per environment and is invisible to ops
const BASE_URL = 'http://localhost:3000';
const API_KEY = 'sk-abc123...';
```

---

## 2. Placement Decision Tree

Before creating anything, **scan the codebase** using the steps in §3. Then follow this tree:

```
Does a constants file/dir already exist for this scope?
│
├── YES — place it there (see §4 for naming conventions)
│
└── NO — how many constants are being added?
    │
    ├── 1–5 values, single topic → create constants.ts in the nearest
    │   relevant directory (co-located with the code that uses them)
    │
    ├── 6+ values OR multiple distinct topics → create a constants/
    │   directory with per-topic files (see §5 for structure)
    │
    └── Values are truly app-wide (used across 3+ unrelated modules)
        → place in src/constants/ or lib/constants/ at project root level
```

**Scope priority (nearest wins):**
```
feature/checkout/constants.ts         ← most specific
features/constants.ts
src/constants/checkout.ts
src/constants/index.ts                ← least specific (last resort)
```

---

## 3. Scanning for Existing Constants

Before writing any new file, run these checks mentally (or with available tools):

1. **Same directory** — does a `constants.ts`, `constants/`, or `*.constants.ts` exist next
   to the file being edited?
2. **Parent directories** — walk up the tree. Check `../constants.ts`, `../../constants/`.
3. **Feature/module scope** — if the file lives in `features/checkout/`, look for
   `features/checkout/constants/` or `features/constants/`.
4. **Project root** — `src/constants/`, `lib/constants/`, `shared/constants/`.
5. **Existing imports** — scan the file's imports for any `constants` path. If found, that
   file is the right home regardless of distance.

If an existing constants location is found, **always prefer it over creating a new file**,
even if adding a new section/group to it. Comment-delimit groups clearly (see §4).

---

## 4. Adding to an Existing Constants File

Group related constants with a heading comment. Keep groups alphabetical within the file.

```ts
// ─── API ──────────────────────────────────────────────────────────────────────

export const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3000';
export const API_VERSION  = 'v2';
export const API_TIMEOUT_MS = 10_000;

// ─── Auth ─────────────────────────────────────────────────────────────────────

export const AUTH_TOKEN_KEY    = 'auth_token';
export const REFRESH_TOKEN_KEY = 'refresh_token';
export const SESSION_TTL_MS    = 60 * 60 * 1_000; // 1 hour

// ─── Pagination ───────────────────────────────────────────────────────────────

export const DEFAULT_PAGE_SIZE = 20;
export const MAX_PAGE_SIZE     = 100;
```

**Naming rules:**
- `SCREAMING_SNAKE_CASE` for all exported constants (TS/JS convention)
- Suffix units: `_MS`, `_PX`, `_REM`, `_SECONDS`, `_MB`
- Prefix booleans: `IS_`, `HAS_`, `SHOULD_`, `ENABLE_`
- Arrays: plural noun — `ALLOWED_ROLES`, `HTTP_METHODS`, `ERROR_CODES`

---

## 5. Creating a New `constants/` Directory

Use a directory when there are multiple distinct topic groups, or when a single
`constants.ts` would exceed ~60–80 lines.

```
feature/checkout/constants/
├── index.ts          ← re-exports everything (barrel file)
├── api.ts            ← endpoint paths, timeouts, versions
├── ui.ts             ← sizes, z-indices, animation durations
├── validation.ts     ← limits, regex patterns, error messages
└── roles.ts          ← domain enums / allowed value sets
```

**`index.ts` barrel (re-export all):**
```ts
export * from './api';
export * from './ui';
export * from './validation';
export * from './roles';
```

Consumers import from the barrel: `import { MAX_FILE_SIZE_MB } from '@/constants'`  
or directly for tree-shaking: `import { MAX_FILE_SIZE_MB } from '@/constants/ui'`

---

## 6. Extraction Patterns by Type

### Numbers
```ts
// Before
await delay(3000);
if (attempts >= 5) fail();

// After — constants/api.ts
export const RETRY_DELAY_MS  = 3_000;
export const MAX_RETRY_COUNT = 5;
```

Use numeric separators (`1_000`, `86_400`) for readability on large numbers.

### Strings
```ts
// Before
if (order.status === 'pending_payment') { ... }

// After — constants/orders.ts
export const ORDER_STATUS = {
  PENDING_PAYMENT: 'pending_payment',
  CONFIRMED:       'confirmed',
  SHIPPED:         'shipped',
  CANCELLED:       'cancelled',
} as const;

export type OrderStatus = typeof ORDER_STATUS[keyof typeof ORDER_STATUS];
```

The `as const` + derived type pattern gives exhaustive type checking without a full enum.

### Repeated literals
```ts
// Before — same string in 4 files
element.setAttribute('data-testid', 'submit-button');

// After — constants/testIds.ts
export const TEST_IDS = {
  SUBMIT_BUTTON:  'submit-button',
  CANCEL_BUTTON:  'cancel-button',
  MODAL_OVERLAY:  'modal-overlay',
} as const;
```

### Constant arrays
```ts
// Before
const valid = ['read', 'write', 'admin'];
if (!valid.includes(scope)) throw ...

// After — constants/permissions.ts
export const PERMISSION_SCOPES = ['read', 'write', 'admin'] as const;
export type PermissionScope = typeof PERMISSION_SCOPES[number];

// Usage — type-safe and self-documenting
if (!PERMISSION_SCOPES.includes(scope as PermissionScope)) throw ...
```

### Env / config values
```ts
// Before
const url = 'https://api.stripe.com/v1';

// After — constants/api.ts (thin wrapper around process.env)
export const STRIPE_API_BASE = process.env.STRIPE_API_BASE ?? 'https://api.stripe.com/v1';
export const STRIPE_API_VERSION = '2024-04-10';
```

Never access `process.env` directly in feature code. Centralise all env reads in one
constants or config file so missing vars fail loudly in one place.

---

## 7. Schema-Driven Types (Zod, Valibot, Arktype, Typebox)

When a runtime validation library is in use, **the schema is the single source of truth**.
TypeScript types must be derived from the schema — never written in parallel.

### The rule: schema first, type derived

```ts
// ❌ — type and schema are two separate sources of truth that can drift
type OrderStatus = 'pending' | 'confirmed' | 'shipped' | 'cancelled';
const orderStatusSchema = z.enum(['pending', 'confirmed', 'shipped', 'cancelled']);

// ✅ — one definition, type derived
const orderStatusSchema = z.enum(['pending', 'confirmed', 'shipped', 'cancelled']);
type OrderStatus = z.infer<typeof OrderStatusSchema>;
```

This applies to every schema shape — objects, enums, unions, literals, arrays.

### Placement: schemas live with their domain constants

Schemas defining a domain's valid values belong in the same constants file as the values
they validate. They are constants themselves.

```ts
// features/orders/constants.ts

// ─── Status ───────────────────────────────────────────────────────────────────

import { z } from 'zod';

export const orderStatusSchema = z.enum(['pending', 'confirmed', 'shipped', 'cancelled']);
export type OrderStatus = z.infer<typeof OrderStatusSchema>;

// Convenience: the values as a plain array (derived from schema, not re-declared)
export const ORDER_STATUSES = OrderStatusSchema.options; // readonly ['pending', ...]
```

### Zod patterns

**Enum (closed string set):**
```ts
export const roleSchema = z.enum(['admin', 'editor', 'viewer']);
export type Role = z.infer<typeof RoleSchema>;
export const ROLES = RoleSchema.options; // derive the array — don't re-declare it
```

**Object / record:**
```ts
export const userSchema = z.object({
  id:    z.string().uuid(),
  email: z.string().email(),
  role:  RoleSchema,
  age:   z.number().int().min(0).max(120),
});
export type User = z.infer<typeof UserSchema>;

// For partials, picks, omits — derive, never hand-write:
export type UserUpdate = z.infer<typeof UserSchema.partial()>;
export type PublicUser = z.infer<typeof UserSchema.omit({ id: true })>;
```

**Discriminated union:**
```ts
export const apiResponseSchema = z.discriminatedUnion('status', [
  z.object({ status: z.literal('ok'),    data: z.unknown() }),
  z.object({ status: z.literal('error'), message: z.string() }),
]);
export type ApiResponse = z.infer<typeof ApiResponseSchema>;
// Never write: type ApiResponse = { status: 'ok'; data: unknown } | { status: 'error'; ... }
```

**Branded / refined types:**
```ts
export const userIdSchema = z.string().uuid().brand('UserId');
export type UserId = z.infer<typeof UserIdSchema>; // string & { __brand: 'UserId' }
```

**Env validation (replaces raw `process.env`):**
```ts
// lib/constants/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NEXT_PUBLIC_API_URL:  z.string().url(),
  STRIPE_SECRET_KEY:    z.string().min(1),
  NODE_ENV:             z.enum(['development', 'test', 'production']),
  PORT:                 z.coerce.number().int().default(3000),
});

// Throws at startup if env is malformed — fail loudly, not silently
export const ENV = EnvSchema.parse(process.env);
export type Env = z.infer<typeof EnvSchema>;
```

### Valibot patterns

```ts
import * as v from 'valibot';

export const roleSchema = v.picklist(['admin', 'editor', 'viewer']);
export type Role = v.InferOutput<typeof RoleSchema>;

export const userSchema = v.object({
  id:   v.pipe(v.string(), v.uuid()),
  role: RoleSchema,
});
export type User = v.InferOutput<typeof UserSchema>;
```

### Arktype patterns

```ts
import { type } from 'arktype';

export const userSchema = type({
  id:   'string.uuid',
  role: "'admin' | 'editor' | 'viewer'",
  age:  'number.integer >= 0',
});
export type User = typeof UserSchema.infer;
```

### Typebox patterns

```ts
import { Type, Static } from '@sinclair/typebox';

export const userSchema = Type.Object({
  id:   Type.String({ format: 'uuid' }),
  role: Type.Union([Type.Literal('admin'), Type.Literal('editor'), Type.Literal('viewer')]),
});
export type User = Static<typeof UserSchema>;
```

### Never do this

```ts
// ❌ Hand-authored type alongside a schema — they will drift
const statusSchema = z.enum(['active', 'inactive']);
type Status = 'active' | 'inactive'; // duplicate — delete it

// ❌ Re-declaring the array the schema already owns
const roleSchema = z.enum(['admin', 'editor', 'viewer']);
const ROLES = ['admin', 'editor', 'viewer'] as const; // use RoleSchema.options instead

// ❌ Using z.string() where z.enum() expresses the constraint
const statusSchema = z.string(); // too loose — loses type narrowing
```

---

## 7a. Object Map vs Flat Exports

Use **flat exports** when values are independent:
```ts
export const MAX_UPLOAD_SIZE_MB = 10;
export const ALLOWED_MIME_TYPES = ['image/png', 'image/jpeg', 'application/pdf'] as const;
```

Use an **object map** (`as const`) when values form a closed domain or will be iterated:
```ts
export const BREAKPOINTS = {
  SM:  640,
  MD:  768,
  LG:  1024,
  XL:  1280,
  '2XL': 1536,
} as const;

// Iterable for responsive logic:
Object.entries(BREAKPOINTS).forEach(([name, px]) => { ... });
```

Avoid TypeScript `enum` — prefer `as const` objects. Enums have footguns (reverse mapping,
tree-shaking issues, const enum pitfalls across module boundaries).

---

## 8. Do / Don't — Schema-Aware Additions

### ✅ Do
- Scan for an existing constants home before creating a new file
- Co-locate constants with the feature that owns them
- Use `as const` + derived type for string/array unions **when no schema library is present**
- When a schema library is present, always use `z.infer<>` / `v.InferOutput<>` / `Static<>` — never hand-write a parallel type
- Use `RoleSchema.options` (Zod) or equivalent to derive constant arrays from the schema
- Validate all `process.env` reads through a schema at startup, not ad-hoc
- Add units to numeric constant names (`_MS`, `_PX`, `_MB`)
- Group with heading comments when adding to a shared file
- Create a barrel `index.ts` when making a `constants/` directory

### ❌ Don't
- Don't create `src/constants/index.ts` for feature-specific values — keep them co-located
- Don't use TypeScript `enum` — use `as const` objects or `z.enum()` instead
- Don't write a `type Foo = 'a' | 'b'` next to a `z.enum(['a', 'b'])` — derive it
- Don't re-declare a constant array that a schema already owns (`.options`, `.enum`)
- Don't name constants `CONSTANT_1`, `VALUE_A` — the name must express intent
- Don't extract a literal that's truly self-evident (`const ONE = 1`, `const EMPTY = ''`)
- Don't duplicate a constant that already exists elsewhere — search first
- Don't hardcode `process.env.NODE_ENV` checks in feature files — centralise in config

---

## 9. When NOT to Extract

Not every literal is magic. Skip extraction when:
- The value is self-evident from context: `array.slice(0, 1)`, `i++`, `x * 2`
- The literal is used exactly once and its meaning is obvious at the call site
- It's a test fixture value that belongs to the test, not the domain
- It's JSX inline style that will never be reused: `style={{ opacity: 0 }}`

The test: *"Would a future reader need to look this up to understand it?"* If no — leave it.

---

## Reference Files

- `references/file-naming-conventions.md` — naming patterns for constants files across common
  project structures (Next.js, Expo, plain Node, monorepo)
- `references/worked-examples.md` — before/after for a real-world feature module with mixed
  magic value types
- `references/schema-examples.md` — worked before/after examples for Zod, Valibot, Arktype,
  and Typebox: enums, objects, discriminated unions, env validation, and co-location patterns

Read these when setting up a new project's constants structure from scratch, or when the user
needs a full worked refactor example.
