# Schema-Driven Types — Worked Examples

## Rule recap

**When a runtime validation library is present, the schema owns the type. Never write a
parallel hand-authored type. Never re-declare an array the schema already owns.**

---

## Example 1: Migrating hand-authored types to Zod-derived types

### Before — type and schema are separate sources of truth

```ts
// features/orders/types.ts
export type OrderStatus = "pending" | "confirmed" | "shipped" | "cancelled";
export type PaymentMethod = "card" | "paypal" | "crypto";

export interface Order {
  id: string;
  status: OrderStatus;
  paymentMethod: PaymentMethod;
  totalCents: number;
  items: { productId: string; qty: number }[];
}

// features/orders/schemas.ts
import { z } from "zod";

// ❌ Values duplicated — already in types.ts
const orderStatusSchema = z.enum([
  "pending",
  "confirmed",
  "shipped",
  "cancelled",
]);
const paymentMethodSchema = z.enum(["card", "paypal", "crypto"]);

export const orderSchema = z.object({
  id: z.string().uuid(),
  status: OrderStatusSchema,
  paymentMethod: PaymentMethodSchema,
  totalCents: z.number().int().min(0),
  items: z.array(
    z.object({
      productId: z.string(),
      qty: z.number().int().min(1),
    }),
  ),
});
```

**Problems:** `orderStatus`, `paymentMethod`, and `order` are all written twice. Adding a new
status requires editing two files. The interface and schema can silently diverge.

---

### After — single source, types derived

```ts
// features/orders/constants.ts
import { z } from "zod";

// ─── Enums ────────────────────────────────────────────────────────────────────

export const orderStatusSchema = z.enum([
  "pending",
  "confirmed",
  "shipped",
  "cancelled",
]);
export type OrderStatus = z.infer<typeof OrderStatusSchema>;
export const ORDER_STATUSES = OrderStatusSchema.options;
// → readonly ['pending', 'confirmed', 'shipped', 'cancelled']

export const paymentMethodSchema = z.enum(["card", "paypal", "crypto"]);
export type PaymentMethod = z.infer<typeof PaymentMethodSchema>;
export const PAYMENT_METHODS = PaymentMethodSchema.options;

// ─── Order ────────────────────────────────────────────────────────────────────

export const orderItemSchema = z.object({
  productId: z.string(),
  qty: z.number().int().min(1),
});

export const orderSchema = z.object({
  id: z.string().uuid(),
  status: OrderStatusSchema,
  paymentMethod: PaymentMethodSchema,
  totalCents: z.number().int().min(0),
  items: z.array(OrderItemSchema),
});

// All types derived — zero hand-authored types
export type OrderItem = z.infer<typeof OrderItemSchema>;
export type Order = z.infer<typeof OrderSchema>;

// Useful partials — still derived, never hand-written
export type OrderUpdate = z.infer<ReturnType<typeof OrderSchema.partial>>;
export type OrderSummary = z.infer<
  ReturnType<
    typeof OrderSchema.pick<{ id: true; status: true; totalCents: true }>
  >
>;
```

Delete `features/orders/types.ts` entirely. Consumers import from `constants.ts`.

---

## Example 2: Env validation with Zod

### Before

```ts
// Scattered across the codebase — no validation, silent failures
const apiUrl = process.env.NEXT_PUBLIC_API_URL;
const port = Number(process.env.PORT) || 3000;
if (process.env.NODE_ENV === 'production') { ... }
```

### After

```ts
// lib/constants/env.ts
import { z } from "zod";

const envSchema = z.object({
  NEXT_PUBLIC_API_URL: z.string().url(),
  NEXT_PUBLIC_APP_ENV: z.enum(["development", "staging", "production"]),
  DATABASE_URL: z.string().min(1),
  JWT_SECRET: z.string().min(32),
  PORT: z.coerce.number().int().default(3000),
  ENABLE_RATE_LIMITING: z.coerce.boolean().default(true),
});

// Throws at module load if any required var is missing/invalid.
// All downstream code gets a fully-typed, validated object.
export const ENV = envSchema.parse(process.env);
export type Env = z.infer<typeof EnvSchema>;

// Usage anywhere in the codebase:
// import { ENV } from '@/lib/constants/env'
// ENV.PORT          → number (not string | undefined)
// ENV.NEXT_PUBLIC_APP_ENV  → 'development' | 'staging' | 'production'
```

---

## Example 3: Discriminated union → derived types

### Before

```ts
// Two parallel definitions — always diverge eventually
type ApiSuccess<T> = { status: "ok"; data: T };
type ApiError = { status: "error"; code: string; message: string };
type ApiResponse<T> = ApiSuccess<T> | ApiError;

const responseSchema = z.union([
  z.object({ status: z.literal("ok"), data: z.unknown() }),
  z.object({
    status: z.literal("error"),
    code: z.string(),
    message: z.string(),
  }),
]);
```

### After

```ts
// constants/api.ts
import { z } from "zod";

export const apiSuccessSchema = z.object({
  status: z.literal("ok"),
  data: z.unknown(),
});

export const apiErrorSchema = z.object({
  status: z.literal("error"),
  code: z.string(),
  message: z.string(),
});

export const apiResponseSchema = z.discriminatedUnion("status", [
  ApiSuccessSchema,
  ApiErrorSchema,
]);

export type ApiSuccess = z.infer<typeof ApiSuccessSchema>;
export type ApiError = z.infer<typeof ApiErrorSchema>;
export type ApiResponse = z.infer<typeof ApiResponseSchema>;
```

---

## Example 4: Valibot — enum with derived array

```ts
// features/auth/constants.ts
import * as v from "valibot";

export const roleSchema = v.picklist(["admin", "editor", "viewer"]);
export type Role = v.InferOutput<typeof RoleSchema>;

// Derive the options array — Valibot exposes them on .options
export const ROLES = RoleSchema.options;
// → ['admin', 'editor', 'viewer']

export const userSchema = v.object({
  id: v.pipe(v.string(), v.uuid()),
  email: v.pipe(v.string(), v.email()),
  role: RoleSchema,
  createdAt: v.date(),
});
export type User = v.InferOutput<typeof UserSchema>;

// Partial for update payloads — derived, not hand-written
export const userUpdateSchema = v.partial(UserSchema);
export type UserUpdate = v.InferOutput<typeof UserUpdateSchema>;
```

---

## Example 5: Arktype — string syntax with derived type

```ts
// features/products/constants.ts
import { type } from "arktype";

export const productStatusSchema = type("'active' | 'draft' | 'archived'");
export type ProductStatus = typeof ProductStatusSchema.infer;

export const productSchema = type({
  id: "string.uuid",
  slug: "string > 0",
  status: "'active' | 'draft' | 'archived'",
  priceCents: "number.integer >= 0",
  tags: "string[]",
});
export type Product = typeof ProductSchema.infer;
```

---

## Example 6: Typebox — JSON Schema compatible

```ts
// features/cms/constants.ts
import { Type, Static } from "@sinclair/typebox";

export const contentStatusSchema = Type.Union([
  Type.Literal("draft"),
  Type.Literal("published"),
  Type.Literal("archived"),
]);
export type ContentStatus = Static<typeof ContentStatusSchema>;

export const contentSchema = Type.Object({
  id: Type.String({ format: "uuid" }),
  title: Type.String({ minLength: 1, maxLength: 200 }),
  status: ContentStatusSchema,
  bodyHtml: Type.String(),
  publishAt: Type.Optional(Type.String({ format: "date-time" })),
});
export type Content = Static<typeof ContentSchema>;

// Partial for patch payloads
export const ContentPatchSchema = Type.Partial(ContentSchema);
export type ContentPatch = Static<typeof ContentPatchSchema>;
```

---

## Detecting when to apply this pattern

Look for any of these signals in the user's code and switch to schema-derived types:

| Signal                                                          | Action                                         |
| --------------------------------------------------------------- | ---------------------------------------------- |
| `import { z } from 'zod'` anywhere in the file or project       | All enums/types must use `z.infer<>`           |
| A `type Foo = 'a' \| 'b'` sitting next to a `z.enum(['a','b'])` | Delete the hand-authored type, derive it       |
| `const VALUES = ['a', 'b'] as const` next to a `z.enum(...)`    | Use `.options` from the schema                 |
| `interface Foo { ... }` mirroring a `z.object({...})`           | Delete the interface, use `z.infer<>`          |
| `process.env.X` accessed directly in feature code               | Move to a `envSchema.parse(process.env)` block |
| `import * as v from 'valibot'`                                  | Use `v.InferOutput<>` for all types            |
| `import { type } from 'arktype'`                                | Use `typeof Schema.infer` for all types        |
| `import { Type, Static } from '@sinclair/typebox'`              | Use `Static<typeof Schema>` for all types      |
