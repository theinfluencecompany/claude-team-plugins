# Anti-Patterns Catalog

Generic examples organized by principle. Each shows the smell, why it violates good engineering, and the fix.

---

## 0. Architecture

> "The goal of software architecture is to minimize the human resources required
> to build and maintain the required system." — Robert C. Martin

### Misaligned module boundaries
```
# BAD — modules organized by technical layer
src/
  controllers/     # all route handlers
  services/        # all business logic
  models/          # all data models
→ Adding a "payments" feature touches all 3 directories. Every PR conflicts.

# GOOD — modules organized by domain
src/
  payments/        # routes + logic + types for payments
  users/           # routes + logic + types for users
  shared/          # truly cross-cutting concerns
→ Adding a "payments" feature is one directory. Isolated change.
```
**Why**: Module boundaries should reflect what changes together. If "add a feature" means touching 5 directories, the boundaries are wrong. The scaling vector is *features* — the structure should bend along that axis.

### Responsibility in the wrong layer
```typescript
// BAD — business logic in the route handler
app.post("/orders", async (c) => {
  const order = c.req.valid("json");
  if (order.total > 10000) await notifyFraudTeam(order);
  const tax = order.total * 0.08;
  const discount = order.coupon ? lookupDiscount(order.coupon) : 0;
  await db.insert("orders", { ...order, tax, discount });
  return c.json({ ok: true });
});

// GOOD — handler is thin, domain logic is testable
app.post("/orders", async (c) => {
  const input = c.req.valid("json");
  const result = await processOrder(input);  // pure domain logic, unit-testable
  return c.json(result);
});
```
**Why**: Route handlers should parse input and return output. Business logic in handlers is untestable without spinning up the whole HTTP stack.

### Wrong decomposition for the scaling vector
```
Problem: System is a monolithic worker handling both sync API requests
and long-running async generation tasks.

Alternative A: Keep monolith, add timeout handling.
  + Simple deployment.  − Long tasks block the worker, can't scale independently.

Alternative B: Split into two services (API + task runner).
  + Each scales independently.  − More infra, more deploy complexity.

Alternative C: Use durable execution (Workflows, Durable Objects, Step Functions).
  + Task state survives crashes, scales independently, single deploy.
  − Vendor lock-in, learning curve.

Converge → depends on the constraint: if the platform already offers durable
execution (e.g., Cloudflare Workflows), C gives the best fit. If not, B.
Never A — mixing sync and async in one process is a scaling trap.
```
**Why**: The diverge-then-converge pattern forces you to articulate tradeoffs instead of going with the first idea. The "best" architecture depends on the actual constraints — audience, platform, scaling vector — not abstract preference.

### Hidden single point of failure
```typescript
// BAD — every module imports and calls this one helper
// If it throws, the entire system goes down. If it changes, every file changes.
import { getConfig } from "../shared/config";
// ... used in 47 files

// GOOD — inject config at construction, modules depend on the shape, not the source
interface AppConfig { apiUrl: string; timeout: number; }
class OrderService {
  constructor(private config: AppConfig) {}
}
```
**Why**: A single shared module imported everywhere is a coupling magnet. If every PR touches it, merge conflicts multiply. If it fails, everything fails. Dependency injection narrows the blast radius.

---

## 1. End-to-End Type Flow

> Types should flow from origin to consumer via derivation. The moment you
> hand-write a type that could be inferred, you've created a second source of
> truth that will rot.

### `any` — a hole in the type boundary
```typescript
// BAD — discards all type information. Bugs enter here undetected.
async function process(client: any): Promise<void> { ... }

// GOOD — derive the chain from the library
type Client = ReturnType<typeof createClient>;
type Session = Awaited<ReturnType<Client["connect"]>>;
async function process(client: Client): Promise<void> { ... }
```
**Why**: If the library changes its return type, derived types update automatically. `any` silently accepts the wrong shape at runtime.

### Hand-written interface that mirrors a schema
```typescript
// BAD — two sources of truth that WILL drift
const userSchema = z.object({ name: z.string(), role: z.enum(["admin", "user"]) });
interface User { name: string; role: "admin" | "user"; }

// GOOD — one origin, one inferred type
const userSchema = z.object({ name: z.string(), role: z.enum(["admin", "user"]) });
type User = z.infer<typeof userSchema>;
```
**Why**: Someone adds a field to the schema, forgets the interface. The mismatch is invisible until runtime. With `z.infer<>`, mismatch is structurally impossible.

### Repeated verbose derivation
```typescript
// BAD — same ReturnType<typeof createClient> in 4 signatures
function send(client: ReturnType<typeof createClient>): void { ... }
function close(client: ReturnType<typeof createClient>): void { ... }

// GOOD — alias once, derive once, use everywhere
type Client = ReturnType<typeof createClient>;
function send(client: Client): void { ... }
function close(client: Client): void { ... }
```

### Ignoring SDK-provided semantics
```typescript
// BAD — raw numeric comparison, ignores the typed boolean
if (result.exitCode !== 0) return handleError();

// GOOD — use the semantic field the SDK already derived for you
if (!result.success) return handleError();
```

### Redundant `as` assertions
```typescript
// BAD — casting papers over a structural type gap
const name = getConfig("name") as string;

// GOOD — narrow with a runtime check at the boundary, let the type follow
const raw = getConfig("name");
if (typeof raw !== "string") throw new TypeError("config.name must be a string");
```

---

## 2. Trust Boundaries

> Validate at the edge where untyped data enters. Trust the interior.

### `process.env` scattered everywhere
```typescript
// BAD — raw env reads scattered across 10 files, each `string | undefined`
function getApiUrl() { return process.env.API_URL || "http://localhost"; }
function getTimeout() { return Number(process.env.TIMEOUT) || 5000; }

// GOOD — validate once at startup, trust the typed config everywhere
const envSchema = z.object({
  API_URL: z.string().url(),
  TIMEOUT: z.coerce.number().default(5000),
});
export const config = envSchema.parse(process.env);
```
**Why**: Env vars are a trust boundary. Parse once into a typed object. Inside the app, `config.API_URL` is `string`, not `string | undefined`.

### Redundant runtime checks inside trusted code
```typescript
// BAD — data already validated by Zod at the route handler
function processOrder(order: Order) {
  if (!order.items || !Array.isArray(order.items)) throw new Error("no items");
  // ... actual logic
}

// GOOD — the type IS the proof
function processOrder(order: Order) {
  // Order type guarantees items: Item[] — just use it
}
```

### API response treated as `any`
```typescript
// BAD — fetch returns unknown data, used with blind property access
const data = await fetch("/api/users").then(r => r.json());
console.log(data.users[0].name); // runtime bomb

// GOOD — validate at the boundary, derive the type
const usersResponse = z.object({ users: z.array(userSchema) });
const data = usersResponse.parse(await fetch("/api/users").then(r => r.json()));
```

---

## 3. Single Source of Truth

> Every piece of knowledge exists in exactly one place.

### Config values defined in multiple places
```yaml
# BAD — same timeout in config.yaml, constants.ts, and the Docker healthcheck
# config.yaml:    timeout: 30
# constants.ts:   const TIMEOUT = 30;
# Dockerfile:     --timeout=30s

# GOOD — one definition, consumed everywhere
# config.yaml:    timeout: 30
# constants.ts:   export const TIMEOUT = config.timeout;  // derived
# Dockerfile:     --timeout=${TIMEOUT}s                   // derived
```

### Imperative mapping where declarative config works
```typescript
// BAD — N lines of if/else for each field
const result: Record<string, string> = {};
if (config.apiKey) result.API_KEY = config.apiKey;
if (config.region) result.REGION = config.region;
// ... 10 more

// GOOD — declarative map + generic loop
const FIELD_MAP = [
  { key: "API_KEY", from: "apiKey" },
  { key: "REGION", from: "region" },
  { key: "DEBUG", from: "debug", fallback: "false" },
] as const;

for (const { key, from, fallback } of FIELD_MAP) {
  result[key] = config[from] ?? fallback ?? "";
}
```

### Magic strings scattered across files
```typescript
// BAD — "completed" appears in handler.ts, poller.ts, and webhook.ts
if (status === "completed") { ... }

// GOOD — single constant, imported everywhere
export const STATUS = { RUNNING: "running", COMPLETED: "completed", FAILED: "failed" } as const;
type Status = typeof STATUS[keyof typeof STATUS];
```

### Cross-layer name drift
```typescript
// BAD — database says `user_id`, API says `userId`, client says `uid`
// Each boundary requires a manual rename — bugs breed in the mapping

// GOOD — one name through the stack, zero mapping, zero drift
```

---

## 4. Abstraction Quality

> The right abstraction eliminates code without hiding intent.

### Generic helper that earns its existence
```typescript
// BAD — same batch-process loop in upload() AND download()
for (let i = 0; i < items.length; i += BATCH_SIZE) {
  await Promise.all(items.slice(i, i + BATCH_SIZE).map(processItem));
}

// GOOD — generic shape, used in both
async function batchMap<T>(
  items: T[], size: number, fn: (item: T) => Promise<void>
): Promise<number> { ... }
```

### Single-use wrapper — don't
```typescript
// BAD — adds indirection for one call site
function formatError(e: Error): string { return e.message; }
return formatError(error);

// GOOD
return error.message;
```

### Premature abstraction for hypothetical future
```typescript
// BAD — generic factory for a single implementation
function createProcessor<T extends Base>(type: string): T { ... }
const proc = createProcessor<OnlyOneType>("default");

// GOOD — just instantiate directly
const proc = new OnlyOneType();
```

---

## 5. Hack Containment & Code Hygiene

> "Sometimes, the elegant implementation is just a function. Not a method.
> Not a class. Not a framework. Just a function." — John Carmack

> One broken window — a single piece of tolerated slop — signals "this codebase
> doesn't care." Research shows existing code smells measurably increase
> developers' tendency to add more. Fix the first window.

### Uncontained hack leaking across modules
```typescript
// BAD — platform workaround leaks into every consumer
// file: user-service.ts
const name = user.displayName || user.name || "Unknown"; // platform returns inconsistent field
// file: order-service.ts
const buyer = order.user.displayName || order.user.name || "Unknown"; // same workaround, copied
// file: notification-service.ts
const recipient = user.displayName || user.name || "Unknown"; // and again

// GOOD — quarantine the hack behind one function, one file
// file: user-utils.ts
/** HACK: Platform returns displayName OR name inconsistently.
 *  Remove when API v3 ships (tracked: PLAT-1234). */
export function getUserName(user: { displayName?: string; name?: string }): string {
  return user.displayName || user.name || "Unknown";
}
// All consumers import getUserName. Hack exists in one place.
```
**Why**: When the platform fixes the inconsistency, you change one function. With the copy-pasted version, you hunt through every file — and miss one.

### Permanent temporary
```typescript
// BAD — "for now" with no removal condition
const timeout = 5000; // TODO: make this configurable for now

// GOOD — either make it configurable now, or accept the constant and drop the TODO
const REQUEST_TIMEOUT_MS = 5000;
```

### Copy-pasted foreign code
```typescript
// BAD — pasted from StackOverflow, half of it is irrelevant
function debounce(func: any, wait: any, immediate: any) {
  var timeout: any;  // var, any, unused `immediate` branch — foreign conventions
  return function() {
    var context = this, args = arguments;
    var later = function() { ... };
    var callNow = immediate && !timeout;
    clearTimeout(timeout);
    timeout = setTimeout(later, wait);
    if (callNow) func.apply(context, args);
  };
}

// GOOD — adapted to the project's conventions, or use a dependency
function debounce<T extends (...args: unknown[]) => void>(
  fn: T,
  delayMs: number
): (...args: Parameters<T>) => void {
  let timer: ReturnType<typeof setTimeout>;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delayMs);
  };
}
```
**Why**: Pasted code brings foreign conventions (`var`, `any`, `this` binding). It signals "I didn't read this, I just dropped it in." Adapt it or use a typed dependency.

### Broken window left unfixed
```typescript
// BAD — you're editing this file and walk past the smell
export async function getUser(id: any) {          // any — existing smell
  const result = await db.query(id);
  return result as any;                            // another any — "was already there"
}

// GOOD — you're in the file, fix it
export async function getUser(id: string): Promise<User> {
  return db.query<User>(id);
}
```
**Why**: "It was already there" is how codebases decay. If you're editing a file, you own its hygiene. Leave it cleaner than you found it.

---

## 6. Dead Code

> Delete the code. Version control remembers.

### Error masking with `|| true`
```bash
# BAD — exit code is always 0, error handling below is dead
result=$(./build.sh 2>&1 || true)
if [ $? -ne 0 ]; then echo "ERROR"; fi  # never reached

# GOOD — let the real exit code propagate
result=$(./build.sh 2>&1)
if [ $? -ne 0 ]; then echo "ERROR"; fi  # actually reachable
```

### Silent catch blocks
```typescript
// BAD — error vanishes
try { await criticalOperation(); } catch {}

// GOOD — if swallowed, say why
try { await criticalOperation(); } catch { /* best-effort: non-fatal cleanup */ }
```

### Unconditional work that should be conditional
```bash
# BAD — reinstalls every startup
npm install

# GOOD — skip if already present
if [ ! -d node_modules ]; then npm install; fi
```

---

## 7. Security

> Never trust input. Never build commands from strings.

### String interpolation into shell commands
```typescript
// BAD — path could contain "; rm -rf /"
await exec(`mkdir -p ${userPath}`);

// GOOD — parameterized SDK methods
await fs.mkdir(userPath, { recursive: true });
```

### Secrets in error messages
```typescript
// BAD
throw new Error(`Auth failed for key ${apiKey}`);

// GOOD
throw new Error("Auth failed: invalid API key");
```

---

## 8. Error Handling

> Handle errors at the level where you have enough context to do something useful.

### Swallowed errors in async operations
```typescript
// BAD — fire-and-forget
someAsyncOperation();

// GOOD — at minimum, log
someAsyncOperation().catch((e) => console.error("[background] failed:", e));
```

### Overly broad try/catch
```typescript
// BAD — catches everything
try {
  const data = parse(input);
  await save(data);
  await notify(data);
} catch { return "something went wrong"; }

// GOOD — narrow scope, propagate what you can't handle
const data = parse(input);
try { await save(data); } catch (e) { throw new SaveError(data.id, e); }
await notify(data);
```
