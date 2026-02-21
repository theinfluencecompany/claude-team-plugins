# TypeScript Type Derivation Reference

> "Every `any` is a lie you tell the compiler. Every hand-written interface that mirrors a schema is a bug waiting to happen. Every `as` is a promise you will break."

**The core law: define the shape once, derive everything else. Zero drift. Zero `any`. Zero hand-written duplicates.**

---

## The Hierarchy

```
BEST:       Derived type (z.infer, ReturnType, $inferSelect)
GOOD:       Branded type (Email, UserId, AccountId)
OK:         unknown + parse at boundary
BAD:        Hand-written interface that mirrors an existing source
UNACCEPTABLE: any
CRIMINAL:   as any
```

Every step down is a choice to disable more of the type system. Stay at the top.

---

## 1. Derive from SDK / Library Types

**The library already did the work. Use it.** If you hand-write an interface that mirrors what the library exports, you have created a second source of truth. It WILL drift.

### Core Extraction Primitives

```typescript
// Return type of a function
type User = ReturnType<typeof createUser>;

// Unwrap async return
type FetchedUser = Awaited<ReturnType<typeof fetchUser>>;

// Function arguments
type UpdateData = Parameters<typeof updateUser>[1];

// Pull from a union
type ClickEvent = Extract<Event, { type: "click" }>;

// Remove from a union
type NonClickEvent = Exclude<Event, { type: "click" }>;

// Chain derivations — the real power
type User = Awaited<ReturnType<typeof getUsers>>[number];
```

### Capture Runtime Values as Types

```typescript
const STATUS = { PENDING: "pending", ACTIVE: "active" } as const;
type StatusValue = (typeof STATUS)[keyof typeof STATUS]; // "pending" | "active"
```

### Where to Find Types

```
node_modules/@prisma/client/index.d.ts   → Prisma namespace, all model types
node_modules/hono/dist/types/index.d.ts  → Hono, Context, Env types
node_modules/zod/lib/types.d.ts          → z.infer, ZodType, ZodSchema
```

The `.d.ts` file is 30 seconds away. Your hand-written type is a liability that will outlive your attention span.

### Library Cheat Sheet

```typescript
// Prisma
type UserWithPosts = Prisma.UserGetPayload<{ include: { posts: true } }>;
const select = { id: true, name: true } satisfies Prisma.UserSelect;
type PublicUser = Prisma.UserGetPayload<{ select: typeof select }>;

// Drizzle
type User = typeof users.$inferSelect;
type NewUser = typeof users.$inferInsert;

// tRPC
type Inputs = inferRouterInputs<AppRouter>;
type CreateUserInput = Inputs["user"]["create"];

// Zod
type User = z.infer<typeof UserSchema>;
type DateInput = z.input<typeof DateSchema>;   // before transform
type DateOutput = z.output<typeof DateSchema>; // after transform

// Hono RPC
type CreateReq = InferRequestType<typeof client.users.$post>["json"];
type UserRes = InferResponseType<typeof client.users.$get, 200>;

// React
type ButtonProps = ComponentProps<"button">;
type InputProps = ComponentProps<typeof CustomInput>;
```

**Always derive.** The only exception: intentionally decoupling from an unstable internal type to create a stable public contract. That's architecture, not laziness.

---

## 2. Schema-First Typing

**The schema is the source of truth. The TypeScript type is `z.infer<>`. There is no third option.**

If you have a Zod schema AND a hand-written interface for the same shape, one of them is wrong — you just don't know which one yet.

```typescript
const UserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(["admin", "user"]).default("user"),
});

type User = z.infer<typeof UserSchema>;  // DERIVED, never hand-written

// Compose and derive — schema-level utilities mirror TS utilities
const AdminSchema = UserSchema.extend({ permissions: z.array(z.string()) });
const UpdateSchema = UserSchema.partial();
const PreviewSchema = UserSchema.pick({ name: true, email: true });

type Admin = z.infer<typeof AdminSchema>;
type UpdateUser = z.infer<typeof UpdateSchema>;
```

### Which Validator

Pick one. Use it everywhere. Do not mix.

| Library | Use When |
|---------|----------|
| **Zod** | Default. Largest ecosystem (tRPC, RHF, Hono). |
| **Valibot** | Bundle-critical (edge, serverless). Up to 95% smaller. |
| **ArkType** | Performance-critical. 3-4x faster than Zod. |
| **TypeBox** | JSON Schema interop (Fastify, OpenAPI). |

Validate at the edge. Trust the interior. Pure internal types that never touch a trust boundary don't need a validator.

---

## 3. Database-First Derivation

**Hand-writing a `User` interface when Prisma/Drizzle already generates one is vandalism.** Generated types are CORRECT by construction. Hand-written interfaces are guesses. Guesses rot.

```typescript
// Prisma — codegen from schema.prisma
import type { Prisma, User } from "@prisma/client";
type UserWithPosts = Prisma.UserGetPayload<{ include: { posts: true } }>;
type CreateUserInput = Prisma.UserCreateInput;

// Drizzle — zero codegen, derives from table definition
export const users = pgTable("users", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
});
type User = typeof users.$inferSelect;
type NewUser = typeof users.$inferInsert;
```

| ORM | Stance |
|-----|--------|
| **Prisma** | Best type safety with relations. Accept the codegen. |
| **Drizzle** | Zero-codegen derivation. SQL-close. |
| **Kysely** | Type-safe raw SQL. `Selectable<T>`, `Insertable<T>`, `Updateable<T>`. |
| **TypeORM** | Legacy only. Use Prisma or Drizzle for new projects. Period. |

If another team owns the database, derive from their API contract (Section 4), not the schema.

---

## 4. API Contract Types

**Writing a client-side interface that "matches" the API response is a bug you haven't found yet.** Derive from the contract.

### tRPC (TS monorepo)

```typescript
// Server defines procedures with Zod input schemas
// Client gets full type inference — zero codegen
const trpc = createTRPCClient<AppRouter>({ links: [...] });
const users = await trpc.user.list.query();  // fully typed

// Derive specific types
type CreateUserInput = inferRouterInputs<AppRouter>["user"]["create"];
```

### Hono RPC

```typescript
// CRITICAL: chain routes for type inference (not separate app.get() calls)
const route = new Hono()
  .get("/users", async (c) => c.json(await db.user.findMany()))
  .post("/users", zValidator("json", CreateUserSchema), async (c) => { ... });

export type AppType = typeof route;

// Client
const client = hc<AppType>("http://localhost:3000");
type CreateReq = InferRequestType<typeof client.users.$post>["json"];
```

**Caveats**: Routes MUST be chained. `tsconfig.json` needs `"strict": true`. Use explicit status codes for status-aware response types.

### OpenAPI / GraphQL

```bash
npx openapi-typescript ./openapi.yaml -o ./src/generated/api.ts  # types only
npx @hey-api/openapi-ts -i ./openapi.yaml -o ./src/generated     # full SDK
```

```typescript
import type { paths, components } from "./generated/api";
type User = components["schemas"]["User"];
type CreateBody = paths["/users"]["post"]["requestBody"]["content"]["application/json"];
```

| Stack | Tool |
|-------|------|
| TS monorepo | **tRPC** — zero codegen, types flow automatically |
| Hono + TS client | **Hono RPC** — chain routes, `InferRequestType`/`InferResponseType` |
| Cross-language / public | **OpenAPI codegen** — generate from spec, never hand-write |
| GraphQL | **graphql-codegen** — generate typed hooks/operations |

**If you're in a TS monorepo fetching JSON and casting with `as`, that's not engineering — it's hoping.**

---

## 5. Utility Types — Mandatory Vocabulary

If you reach for a hand-written interface when `Pick`, `Omit`, or `ReturnType` would work, you are creating a maintenance liability.

```typescript
interface User { id: string; name: string; email: string; password: string; createdAt: Date }

type UserPreview = Pick<User, "id" | "name">;
type PublicUser = Omit<User, "password">;
type UserUpdate = Partial<Omit<User, "id" | "createdAt">>;
type FrozenUser = Readonly<User>;
type UserMap = Record<string, User>;
type DefiniteUser = NonNullable<User | null>;
```

### `satisfies` over `as` — Always

```typescript
// as — widens and lies
const config: Record<string, string | number> = { port: 3000 };
config.port; // string | number — WRONG, we know it's a number

// satisfies — validates and preserves
const config = { port: 3000, host: "localhost" } satisfies Record<string, string | number>;
config.port; // 3000 — literal preserved
```

### `as const` for Literal Values

```typescript
const STATUS = { PENDING: "pending", ACTIVE: "active" } as const;
type Status = (typeof STATUS)[keyof typeof STATUS]; // "pending" | "active"
```

### Discriminated Unions for Mutually Exclusive States

```typescript
// BAD: 2^3 = 8 states, only 4 are valid
interface State { isLoading: boolean; data: User | null; error: Error | null }

// GOOD: exactly 4 states, all valid
type State =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: Error };
```

Boolean flags that create impossible combinations are bugs encoded in types.

**Restraint**: Conditional types, `infer`, template literals, and deep mapped types belong in library code, not application code. If your type gymnastics nest 3+ levels, simplify the data model.

---

## 6. Architecture Patterns

### Parse, Don't Validate (Alexis King)

The single most important idea in typed programming. Validation discards the proof. Parsing encodes it.

```typescript
// BAD: validated but proof is lost — input is still `string`
function processEmail(input: string) {
  if (!isValidEmail(input)) throw new Error("Invalid");
  sendEmail(input); // any string accepted here
}

// GOOD: proof encoded in the type
type Email = string & { readonly __brand: "Email" };
function parseEmail(input: string): Email {
  if (!isValidEmail(input)) throw new Error("Invalid");
  return input as Email;
}
function sendEmail(to: Email) {} // ONLY parsed Email accepted
```

### Brand Your Domain IDs

`string` is not a type — it is the ABSENCE of a type. If `transferMoney(from: string, to: string)` swaps arguments, the compiler shrugs. One line per brand. Catches argument-swap bugs at compile time.

```typescript
type UserId = string & { readonly __brand: "UserId" };
type AccountId = string & { readonly __brand: "AccountId" };
function UserId(id: string): UserId { return id as UserId; }
function AccountId(id: string): AccountId { return id as AccountId; }

transferMoney(accountId, accountId, 100);  // OK
transferMoney(userId, accountId, 100);     // COMPILE ERROR
```

### Schema -> Infer -> Propagate

```typescript
// 1. SCHEMA: single source of truth
const UserSchema = z.object({ id: z.string().uuid(), name: z.string(), email: z.string().email(), role: z.enum(["admin", "user"]) });

// 2. INFER
type User = z.infer<typeof UserSchema>;

// 3. PROPAGATE — all variations derived
type CreateUserInput = Omit<User, "id">;
type UpdateUserInput = Partial<Pick<User, "name" | "email" | "role">>;
type PublicUser = Omit<User, "role">;

// Change the schema → everything updates. Zero drift.
```

### `any` is Forbidden. `as` is Suspect.

`any` means "I am turning off the type checker for this value AND everything downstream." It is viral. **Zero excuse in application code.** Read the `.d.ts`, use `unknown`, or parse with a schema.

`as` is a weaker form. The only acceptable uses: DOM types and branded-type constructors. Everything else: parse it, narrow it, derive it.

---

## 7. Anti-Patterns — These Are Bugs

Every pattern here is a type-system hole that WILL produce a runtime bug. Not "might."

### 7.1 Schema + Separate Interface — BUG
Two definitions of the same shape. When one changes, the other silently drifts. Delete the interface. Use `z.infer<>`.

### 7.2 Using `any` — BUG
Not "I'll type this later." It is permanently disabling the checker for this value and everything downstream. Grep for `any` now. Every hit is a latent bug.

### 7.3 `as` Casting — SMELL (often BUG)
Every `as` is you overriding the compiler. Parse it. Narrow it. Derive it. Don't cast it.

### 7.4 Redundant Runtime Checks Inside Validated Code — SMELL
If you validated at the boundary and then check `typeof` inside, you don't trust your own code. The type IS the proof.

### 7.5 Same Shape in Multiple Files — BUG
`UserResponse` in handlers, `UserData` in services, `UserProps` in components — all `{ id: string; name: string; email: string }`. One definition, imported everywhere. Derive variations with `Pick`/`Omit`.

---

## 8. Discovering SDK Types

**Before writing ANY type by hand, check whether it already exists.** Step zero.

| Need | Tool |
|------|------|
| What types a package exports | Read `node_modules/pkg/index.d.ts` |
| Jump to a type definition | IDE `Cmd+Click` → Go to Definition |
| What types your code produces | `npx tsc --declaration --emitDeclarationOnly` |
| Test type derivations | [TypeScript Playground](https://www.typescriptlang.org/play) |
| Common utility types | `type-fest` (`Simplify`, `SetRequired`, `Merge`, `PartialDeep`) |

---

## Quick Decision Guide

| Situation | Correct | WRONG |
|-----------|---------|-------|
| Type from database | `Prisma.GetPayload<>` / `$inferSelect` | Hand-written `interface User` |
| Type from schema | `z.infer<typeof Schema>` | Separate `interface` |
| Type from function return | `Awaited<ReturnType<typeof fn>>` | Copying the shape by hand |
| Type from API contract | `inferRouterOutputs` / codegen | Client-side `interface` |
| Subset of properties | `Pick<Base, "a" \| "b">` / `Omit` | Re-typing fields |
| All fields optional | `Partial<Base>` | New interface with `?` |
| Two string IDs confused | Branded types | Both as `string` |
| Mutually exclusive states | Discriminated union | Boolean flags |
| Config shape check | `satisfies` | `as` (widens and lies) |
| External data enters | `unknown` + parse | `any` + cast |

---

## Sources

- [Total TypeScript: Deriving Types](https://www.totaltypescript.com/books/total-typescript-essentials/deriving-types)
- [Alexis King: Parse, Don't Validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
- [Sandi Metz: Make Illegal States Unrepresentable](https://khalilstemmler.com/articles/typescript-domain-driven-design/make-illegal-states-unrepresentable/)
- [Effective TypeScript (2025)](https://effectivetypescript.com/)
- [Effect-TS: Branded Types](https://effect.website/docs/code-style/branded-types/)
- [Zod Documentation](https://zod.dev/)
- [Drizzle ORM: Type Inference](https://orm.drizzle.team/docs/goodies)
- [Prisma: Partial Structures](https://www.prisma.io/docs/orm/prisma-client/type-safety/operating-against-partial-structures-of-model-types)
- [tRPC Documentation](https://trpc.io/)
- [Hono RPC Guide](https://hono.dev/docs/guides/rpc)
- [openapi-typescript](https://github.com/openapi-ts/openapi-typescript)
- [type-fest](https://github.com/sindresorhus/type-fest)
