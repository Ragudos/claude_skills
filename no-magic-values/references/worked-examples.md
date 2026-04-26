# Worked Examples — Before & After

## Example 1: API service with mixed magic values

### Before

```ts
// features/users/api.ts
export async function fetchUsers(page: number) {
  const res = await fetch(
    `https://api.example.com/v2/users?page=${page}&limit=25`,
    {
      headers: { "X-Api-Key": "abc123secret" },
      signal: AbortSignal.timeout(8000),
    },
  );

  if (!res.ok) {
    if (res.status === 429) throw new Error("RATE_LIMITED");
    if (res.status === 401) throw new Error("UNAUTHORIZED");
    throw new Error("FETCH_FAILED");
  }

  return res.json();
}

export async function createUser(data: unknown) {
  const res = await fetch("https://api.example.com/v2/users", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Api-Key": "abc123secret",
    },
    body: JSON.stringify(data),
    signal: AbortSignal.timeout(8000),
  });
  return res.json();
}
```

**Magic values found:**

- `'https://api.example.com/v2'` — repeated base URL (env value)
- `'abc123secret'` — hardcoded API key (env value)
- `25` — magic page size
- `8000` — magic timeout
- `429`, `401` — magic HTTP status codes
- `'RATE_LIMITED'`, `'UNAUTHORIZED'`, `'FETCH_FAILED'` — magic error codes

**Scan result:** No existing `constants` file in `features/users/`. Multiple distinct topics
(API config, HTTP codes, error codes) → create `features/users/constants/`.

### After — `features/users/constants/api.ts`

```ts
export const USERS_API_BASE = `${process.env.NEXT_PUBLIC_API_URL ?? ""}/v2/users`;
export const USERS_API_KEY = process.env.USERS_API_KEY ?? "";
export const USERS_PAGE_SIZE = 25;
export const USERS_TIMEOUT_MS = 8_000;
```

### After — `features/users/constants/errors.ts`

```ts
export const USER_ERROR_CODES = {
  RATE_LIMITED: "RATE_LIMITED",
  UNAUTHORIZED: "UNAUTHORIZED",
  FETCH_FAILED: "FETCH_FAILED",
} as const;

export type UserErrorCode =
  (typeof USER_ERROR_CODES)[keyof typeof USER_ERROR_CODES];
```

### After — `features/users/constants/http.ts`

_(Or reference a shared `src/constants/http.ts` if one exists)_

```ts
export const HTTP_STATUS = {
  UNAUTHORIZED: 401,
  TOO_MANY_REQS: 429,
} as const;
```

### After — `features/users/constants/index.ts`

```ts
export * from "./api";
export * from "./errors";
export * from "./http";
```

### After — `features/users/api.ts`

```ts
import {
  USERS_API_BASE,
  USERS_API_KEY,
  USERS_PAGE_SIZE,
  USERS_TIMEOUT_MS,
  HTTP_STATUS,
  USER_ERROR_CODES,
} from "./constants";

export async function fetchUsers(page: number) {
  const res = await fetch(
    `${USERS_API_BASE}?page=${page}&limit=${USERS_PAGE_SIZE}`,
    {
      headers: { "X-Api-Key": USERS_API_KEY },
      signal: AbortSignal.timeout(USERS_TIMEOUT_MS),
    },
  );

  if (!res.ok) {
    if (res.status === HTTP_STATUS.TOO_MANY_REQS)
      throw new Error(USER_ERROR_CODES.RATE_LIMITED);
    if (res.status === HTTP_STATUS.UNAUTHORIZED)
      throw new Error(USER_ERROR_CODES.UNAUTHORIZED);
    throw new Error(USER_ERROR_CODES.FETCH_FAILED);
  }

  return res.json();
}

export async function createUser(data: unknown) {
  const res = await fetch(USERS_API_BASE, {
    method: "POST",
    headers: { "Content-Type": "application/json", "X-Api-Key": USERS_API_KEY },
    body: JSON.stringify(data),
    signal: AbortSignal.timeout(USERS_TIMEOUT_MS),
  });
  return res.json();
}
```

---

## Example 2: Auth module — adding to an existing constants file

### Situation

`features/auth/constants.ts` already exists with token keys. New code adds a login form
with hardcoded validation limits.

### New code (magic values highlighted)

```ts
// features/auth/LoginForm.tsx
function validate(email: string, password: string) {
  if (email.length > 254) return "Email too long";
  if (password.length < 8) return "Password too short";
  if (password.length > 128) return "Password too long";
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) return "Invalid email";
}

const ALLOWED_PROVIDERS = ["google", "github", "email"];
```

**Scan result:** `features/auth/constants.ts` exists → add to it, don't create a new file.

### After — appended to `features/auth/constants.ts`

```ts
// ─── (existing content) ───────────────────────────────────────────────────────

export const AUTH_TOKEN_KEY = "auth_token";
export const REFRESH_TOKEN_KEY = "refresh_token";
export const SESSION_TTL_MS = 60 * 60 * 1_000;

// ─── Validation ───────────────────────────────────────────────────────────────

export const EMAIL_MAX_LENGTH = 254; // RFC 5321
export const PASSWORD_MIN_LENGTH = 8;
export const PASSWORD_MAX_LENGTH = 128;
export const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

// ─── Providers ────────────────────────────────────────────────────────────────

export const AUTH_PROVIDERS = ["google", "github", "email"] as const;
export type AuthProvider = (typeof AUTH_PROVIDERS)[number];
```

### After — `features/auth/LoginForm.tsx`

```ts
import {
  EMAIL_MAX_LENGTH,
  PASSWORD_MIN_LENGTH,
  PASSWORD_MAX_LENGTH,
  EMAIL_REGEX,
  AUTH_PROVIDERS,
  type AuthProvider,
} from "./constants";

function validate(email: string, password: string) {
  if (email.length > EMAIL_MAX_LENGTH) return "Email too long";
  if (password.length < PASSWORD_MIN_LENGTH) return "Password too short";
  if (password.length > PASSWORD_MAX_LENGTH) return "Password too long";
  if (!EMAIL_REGEX.test(email)) return "Invalid email";
}
```

---

## Example 3: Constant array → typed union

### Before

```ts
// Used in 3 different files:
const tiers = ["free", "pro", "enterprise"];
if (!["free", "pro", "enterprise"].includes(user.tier)) redirect("/upgrade");
type Tier = "free" | "pro" | "enterprise"; // duplicated manually
```

### After — `features/billing/constants.ts`

```ts
export const SUBSCRIPTION_TIERS = ["free", "pro", "enterprise"] as const;
export type SubscriptionTier = (typeof SUBSCRIPTION_TIERS)[number];

// Guard function — use instead of .includes() for type narrowing
export function isValidTier(value: unknown): value is SubscriptionTier {
  return SUBSCRIPTION_TIERS.includes(value as SubscriptionTier);
}
```

### After — usage

```ts
import {
  SUBSCRIPTION_TIERS,
  isValidTier,
  type SubscriptionTier,
} from "../billing/constants";

if (!isValidTier(user.tier)) redirect("/upgrade");
// user.tier is now narrowed to SubscriptionTier ✓
```

---

## Example 4: UI constants — when to stay flat vs use a directory

### Flat (few values, single concern)

```ts
// components/Modal/constants.ts
export const MODAL_Z_INDEX = 1000;
export const MODAL_BACKDROP_OPACITY = 0.5;
export const MODAL_ANIMATION_MS = 200;
export const MODAL_MAX_WIDTH_PX = 640;
```

### Directory (large component system with many concerns)

```
components/DataTable/constants/
├── index.ts
├── pagination.ts    → PAGE_SIZE_OPTIONS, DEFAULT_PAGE_SIZE, MAX_VISIBLE_PAGES
├── sorting.ts       → SORT_DIRECTIONS, DEFAULT_SORT_FIELD
├── columns.ts       → MIN_COLUMN_WIDTH_PX, DEFAULT_COLUMN_WIDTH_PX, FROZEN_COLUMN_Z_INDEX
└── accessibility.ts → ARIA_LABELS, KEYBOARD_SHORTCUTS
```

**Rule of thumb:** if you find yourself scrolling past 3 heading comments in one file, it's
time to split into a directory.
