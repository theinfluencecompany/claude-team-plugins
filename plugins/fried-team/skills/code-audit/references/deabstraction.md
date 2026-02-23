# Deabstraction: Kill Your Layers

> "Every layer of indirection is a tax on every future reader. Most codebases are bankrupt."

**The core law: code that does nothing — delegates without transforming, wraps without adding, exists because "that's the pattern" — is not architecture. It is dead weight. Delete it.**

```
BEST:        Direct call. One file. Obvious.
GOOD:        Abstraction with 3+ callers, zero flags, genuine shared logic.
SUSPECT:     Wrapper that "might be useful later." It won't be.
BAD:         Delegation chain: A calls B calls C, B does nothing.
UNACCEPTABLE: Wrong abstraction with accumulated boolean flags and mode switches.
CRIMINAL:    "Enterprise architecture" — Manager calls Handler calls Processor calls Engine.
```

---

## 1. Principles — Why Deabstract

### The Wrong Abstraction Is Worse Than No Abstraction (Sandi Metz)

> "Duplication is far cheaper than the wrong abstraction."

A wrong abstraction doesn't just fail to help — it actively poisons the codebase. Every developer adds another boolean flag, another conditional, another "mode." The abstraction becomes a Frankenstein that nobody understands and everybody is afraid to touch. **The fix is not to "improve" the abstraction. The fix is to inline it back into its callers and start over.**

**The death spiral:**
1. Programmer A sees duplication, extracts an abstraction.
2. New requirement arrives that *almost* fits.
3. Programmer B adds a parameter and conditional instead of questioning the abstraction.
4. Repeat 5 times. The abstraction is now incomprehensible.
5. Everyone is afraid to touch it. Velocity halves.

**Diagnostic:** Boolean flags? Type switches? "Mode" parameters? **It is the wrong abstraction. Kill it.**

### Three Lines > One Premature Helper (Rule of Three)

Two instances of similar code do NOT justify abstraction. You do not have enough information yet. Wait for the THIRD occurrence — only then do you know what truly varies and what stays the same. Abstracting at two is guessing. Guessing creates the wrong abstraction. The wrong abstraction is worse than duplication. See above.

### YAGNI — You Aren't Gonna Need It (Kent Beck)

> "The best code is the code you never write."

"We might need this later" is the most expensive sentence in software. Every speculative abstraction, every "just in case" layer, every "future-proofed" interface — they all cost maintenance NOW for a future that almost never arrives. Before adding ANY capability: "Do I need this RIGHT NOW for a REAL use case?" If the answer is anything other than an emphatic yes, **don't build it.**

### Optimize for Change, Not for DRY (AHA — Kent C. Dodds)

DRY is not a virtue. DRY is a tool. When applied dogmatically it produces the wrong abstraction (see Law 1). Tolerate duplication until the pattern **screams** for extraction. Three similar blocks that you understand instantly are infinitely better than one "clever" abstraction that requires 10 minutes to trace.

---

## 2. The Diagnostic — How to Identify

Run this BEFORE touching code. The diagnostic tells you what to kill. The patterns in §3 and §4 tell you how.

### The Call-Depth Audit

**Count the hops.** For any user-facing operation, trace from entry point to side effect (DB write, API call, file write). Every hop that doesn't transform, validate, or handle errors is a candidate for deletion.

```
HEALTHY:    Route → Handler (validates + does work + responds)       = 1 hop
ACCEPTABLE: Route → Handler → Repository (real DB abstraction)      = 2 hops
SUSPECT:    Route → Controller → Service → Repository               = 3 hops
DISEASED:   Route → Controller → Service → Manager → Repository → DAO = 5 hops
```

**The rule: if you can't justify what each hop DOES (not what it IS), inline it.**

### The Per-Hop Test

At each hop in the call chain, ask:

```
Transform data shape?    → KEEP
Validate/parse input?    → KEEP
Handle errors uniquely?  → KEEP
Orchestrate 2+ calls?    → KEEP
Just forward arguments?  → INLINE
Just rename the function? → INLINE
```

**Surviving hops: 1–2 healthy. 3 must be justified. 4+ diseased — deabstract immediately.**

### The Kill/Keep Verdict

**Kill it** (any ONE signal = guilty):

| Signal | Verdict |
|--------|---------|
| Solves no current, concrete problem | YAGNI — delete |
| Cannot explain what it DOES (only what it IS) | Pure indirection — inline |
| Removing it breaks no tests | Doing nothing — delete |
| Accumulated boolean flags or mode parameters | Wrong Abstraction — inline and re-extract |
| Understanding one behavior requires 3+ files | LoB violation — colocate or inline |
| Just calls another function/class | Pass-through — inline |
| Built because "we might need it" or "that's the pattern" | Architecture Astronaut — delete |

**Keep it** (must satisfy 2+):

| Signal | Why |
|--------|-----|
| Represents a genuine domain concept | Names matter |
| Multiple callers use it the SAME way (zero flags) | Real shared logic |
| Removing it would duplicate complex logic | Earned its place |
| Enforces an invariant callers would violate | Guardrail |
| Hides complexity callers genuinely don't need | Real encapsulation |
| Has been stable over a meaningful period | Battle-tested |

**If it doesn't satisfy at least 2 from the "Keep" column — it doesn't deserve to exist.**

---

## 3. Code-Level Patterns — Fixing Functions and Structure

Small-scale removals. One file, one function, one expression at a time.

### 3.1 Inline Function — The Default Refactoring

If a function's body is as clear as its name, the function is not an abstraction — it is a detour. **Inline it.**

```typescript
// BEFORE — pointless indirection
function getRating(driver: Driver) {
  return moreThanFiveLateDeliveries(driver) ? 2 : 1;
}
function moreThanFiveLateDeliveries(driver: Driver) {
  return driver.numberOfLateDeliveries > 5;
}

// AFTER — direct, obvious, one less function to maintain
function getRating(driver: Driver) {
  return driver.numberOfLateDeliveries > 5 ? 2 : 1;
}
```

**Inline when:** Body is a single expression. Name adds zero explanatory value. A calls B calls C where B and C do nothing.

### 3.2 Inline Class — Gutted Shells Must Die

A class with no behavior is not a class — it is a struct wearing a costume. A class gutted by previous refactorings is a ghost occupying a file. **Fold it into its consumer.** If two classes need merging before re-extraction along better boundaries, inline both first.

### 3.3 Kill Your Darlings — Cleverness Is a Liability

> "Everyone knows that debugging is twice as hard as writing a program. So if you're as clever as you can be when you write it, how will you ever debug it?" — Kernighan

If you feel *proud* of a design, that is a red flag. Would a new team member understand this in 5 minutes? Is there a boring, obvious alternative? **Use the boring one.**

```typescript
// BEFORE — "elegant" monad pipeline for... checking if a user is active
const isActive = compose(chain(validateSession), map(extractToken), fromNullable)(user);

// AFTER — boring, obvious, debuggable
const isActive = user != null && user.session && !user.session.expired;
```

Cleverness that doesn't earn its complexity is vanity.

### 3.4 Flat Over Nested — Every Indent Is Cognitive Debt

Nesting is a tax on the reader's working memory. Guard clauses and early returns flatten the pyramid and let the happy path read top-to-bottom.

```typescript
// BEFORE — nested pyramid of doom
function process(data: Data) {
  if (data != null) {
    if (data.isValid()) {
      if (data.hasPermission()) {
        return doWork(data);
      } else { throw new PermissionError(); }
    } else { throw new ValidationError(); }
  } else { throw new ValueError("No data"); }
}

// AFTER — flat, scannable, obvious
function process(data: Data) {
  if (data == null) throw new ValueError("No data");
  if (!data.isValid()) throw new ValidationError();
  if (!data.hasPermission()) throw new PermissionError();
  return doWork(data);
}
```

### 3.5 Direct Style Over Wrapper Chains

Promise chains, callback pyramids, monadic wrappers — if they add ceremony without benefit, they are noise. Write sequential code. Let `async/await` do its job.

```typescript
// BEFORE — .then() chain
getUser(id)
  .then(user => getOrders(user.id))
  .then(orders => getItems(orders[0].id))
  .then(items => render(items))
  .catch(handleError);

// AFTER — direct, debuggable, breakpointable
const user = await getUser(id);
const orders = await getOrders(user.id);
const items = await getItems(orders[0].id);
render(items);
```

### 3.6 Composition Over Inheritance — Always

Deep inheritance trees are the wrong abstraction at the language level. They create rigid coupling where changing a base class shatters every descendant. **Compose from small, focused pieces.** If you inherit from more than one level deep, you are building a house of cards.

---

## 4. Architectural Patterns — Fixing Layers and Boundaries

Large-scale removals. Multiple files, entire layers, system-level indirection.

### 4.1 The Service Layer That Does Nothing

The most common architectural lie. A "service" that takes arguments from the controller and passes them to the repository unchanged. It exists because "separation of concerns" — but there are no concerns being separated.

```typescript
// BEFORE — the do-nothing service (3 files, 3 hops)
// controller.ts
export const createUser = async (c: Context) => {
  const body = await c.req.json();
  const user = await userService.create(body);  // hop 1
  return c.json(user);
};
// service.ts
export const userService = {
  create: (data: CreateUser) => userRepo.insert(data),  // hop 2 — does NOTHING
};
// repo.ts
export const userRepo = {
  insert: (data: CreateUser) => db.insert(users).values(data).returning(),
};

// AFTER — direct (1 file, 1 hop)
export const createUser = async (c: Context) => {
  const body = await c.req.json();
  const user = await db.insert(users).values(body).returning();
  return c.json(user);
};
```

**"But what if we need business logic later?"** Then add it later. YAGNI. The service layer takes 30 seconds to introduce when you actually need it. Maintaining it for months "just in case" costs orders of magnitude more.

**Earns its place:** Orchestrates multiple repositories in a transaction, applies business rules, or transforms data between domain and persistence shapes. If it's `(data) => repo.doThing(data)` — **kill it.**

### 4.2 The Repository Pattern Over an ORM

Prisma IS a repository. Drizzle IS a repository. Wrapping them in another repository is wrapping a wrapper. You are not adding abstraction — you are adding indirection.

```typescript
// BEFORE — wrapping a wrapper
class UserRepository {
  async findById(id: string) {
    return this.prisma.user.findUnique({ where: { id } });  // just forwarding
  }
  async create(data: CreateUser) {
    return this.prisma.user.create({ data });                // just forwarding
  }
}

// AFTER — use the ORM directly, it already IS the repository
const user = await prisma.user.findUnique({ where: { id } });
```

**"But what if we switch databases?"** You won't. And if you do, the repository wrapper won't save you — the query shapes, transaction semantics, and performance characteristics are all different. You'll rewrite everything anyway.

**Earns its place:** Encapsulates complex queries (multi-join, aggregation, raw SQL) behind a domain-meaningful name. `userRepo.findActiveWithRecentOrders()` hiding a 15-line query — that's a real abstraction. `userRepo.findById(id)` wrapping `prisma.user.findUnique({ where: { id } })` — that's a crime.

### 4.3 The Controller/Handler Split

In frameworks like Hono, Express, or Fastify, the route handler IS the controller. Extracting "controllers" into separate files that the route file imports and calls adds a hop with zero benefit.

```typescript
// BEFORE — pointless controller extraction
// routes.ts
app.post("/users", userController.create);
// controller.ts
export const userController = {
  create: async (c: Context) => {
    const body = await c.req.json();
    const user = await createUser(body);
    return c.json(user, 201);
  },
};

// AFTER — the route IS the controller
app.post("/users", async (c) => {
  const body = await c.req.json();
  const user = await createUser(body);
  return c.json(user, 201);
});
```

**Earns its place:** Same handler logic reused across multiple routes or protocols (HTTP + WebSocket + CLI). That almost never happens.

### 4.4 The Factory That Creates One Thing

A factory exists to choose between implementations. If there is ONE implementation, the factory is a function call that always returns the same thing. That is not a pattern — it is a ritual.

```typescript
// BEFORE — factory for one implementation
function createNotificationService(type: "email"): EmailService;
function createNotificationService(type: string) {
  switch (type) {
    case "email": return new EmailService();
    default: throw new Error("Unknown type");
  }
}
const service = createNotificationService("email");

// AFTER — just use the thing
const service = new EmailService();
```

**Earns its place:** 2+ implementations selected at runtime. One implementation = direct construction. Period.

### 4.5 The DTO Mapping Chain

Entity → DTO → ViewModel → Response. Four representations of the same data. Three mapping functions that shuffle fields. If the shapes are identical (or nearly so), you have three layers of `{ ...obj, field: obj.field }` that exist only to satisfy an architecture diagram.

```typescript
// BEFORE — three transformations that do nothing
const entity = await db.query.users.findFirst({ where: eq(users.id, id) });
const dto = toUserDTO(entity);          // { id, name, email } → { id, name, email }
const viewModel = toUserViewModel(dto); // { id, name, email } → { id, name, email }
return c.json(toUserResponse(viewModel)); // { id, name, email } → { id, name, email }

// AFTER — select what you need, return it
const user = await db.query.users.findFirst({
  where: eq(users.id, id),
  columns: { id: true, name: true, email: true },
});
return c.json(user);
```

**Earns its place:** API response shape genuinely differs from the database shape — computed fields, nested relations flattened, sensitive fields stripped. If the mapper is `(entity) => ({ ...entity })` — **delete it.**

### 4.6 The Event Bus for Two Subscribers

Pub/sub is powerful when you have N unknown subscribers. When you have 2 known subscribers, the event bus is a `function call` with extra steps and zero type safety.

```typescript
// BEFORE — event bus indirection
eventBus.emit("user.created", user);
// subscriber1.ts — registered somewhere in a boot file
eventBus.on("user.created", sendWelcomeEmail);
// subscriber2.ts
eventBus.on("user.created", createDefaultWorkspace);

// AFTER — direct calls, type-safe, traceable, debuggable
await sendWelcomeEmail(user);
await createDefaultWorkspace(user);
```

**Earns its place:** Subscribers genuinely unknown at compile time (plugin systems, multi-tenant hooks), guaranteed delivery needed (real queue like SQS/BullMQ, not in-memory EventEmitter), or decoupling required across service boundaries. An in-memory EventEmitter with 2 hardcoded `.on()` calls is not decoupling — it is obfuscation.

### 4.7 Middleware — Guilty Until Proven Innocent

Pass-through middleware is the most common form of unnecessary abstraction. It exists because someone read a "clean architecture" blog, not because it solves a problem.

**Every middleware must answer YES to at least one:**
- Does it transform the request or response?
- Does it handle errors differently from the layers above or below?
- If removed, would any test fail?

**If NO to all three — delete it.** It is not architecture. It is bureaucracy.

**Legitimate:** Authentication, rate limiting, request validation, circuit breaking, observability. These do real work.

**Illegitimate:** "Service layers" that call repositories with zero transformation. "Controller layers" that call services with zero logic. "Middleware" that logs and passes through. **Delete all of them.**

---

## 5. The Removal Sequence

When an abstraction is sentenced to removal:

1. **Identify all callers.**
2. **Inline** into each caller. No shared function means no shared bugs.
3. **Delete dead branches** — each inlined copy has conditions that only other callers needed.
4. **Simplify each caller** independently for its own context.
5. **Wait.** Do NOT immediately re-extract. Let the code breathe.
6. **Apply Rule of Three** — only re-extract when the THIRD real instance appears.

### Code Organization After Removal

Once layers are removed, the surviving code must be organized well:

- **Locality of Behavior** (Carson Gross) — The behavior of a unit of code should be obvious by reading only that unit. If understanding one behavior requires 3+ files, the abstraction boundaries are drawn in the wrong place. **The reader should never play detective.**

- **Colocation** — Things that change together live together. Code + tests + styles + types for one feature belong in one directory. Scattering by file type is not organization — it is fragmentation.

- **Pit of Success** (Rico Mariani) — Design APIs so the easiest path IS the correct path. If you need defensive middleware to prevent wrong usage, the API is wrong. **Make the right thing the easy thing.**

---

## 6. Warning Signs — The Architecture Astronaut (Joel Spolsky)

> "When you go too far up, abstraction-wise, you run out of oxygen."

Your codebase has an Architecture Astronaut problem when:

- The architecture diagram has more boxes than features.
- A new developer needs a week to understand the "framework" before writing business logic.
- The system can theoretically handle requirements it will never face.
- Class names include: "Manager," "Handler," "Processor," "Engine," "Framework," "Platform," "Bus."
- Adding a simple feature requires touching 7 files across 4 layers.
- There are more interfaces than implementations.

**The counter-test that kills astronauts: "What concrete problem does this solve TODAY?"**

If the answer starts with "well, in the future..." — delete it.

---

## Sources

- [Sandi Metz: The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction)
- [Kent C. Dodds: AHA Programming](https://kentcdodds.com/blog/aha-programming)
- [Kent Beck: XP — YAGNI](https://wiki.c2.com/?YouArentGonnaNeedIt)
- [Martin Fowler: Refactoring — Inline Function/Class](https://refactoring.com/catalog/)
- [Joel Spolsky: Don't Let Architecture Astronauts Scare You](https://www.joelonsoftware.com/2001/04/21/dont-let-architecture-astronauts-scare-you/)
- [Carson Gross: Locality of Behavior](https://htmx.org/essays/locality-of-behaviour/)
- [Rico Mariani: Pit of Success](https://blog.codinghorror.com/falling-into-the-pit-of-success/)
- [Brian Kernighan & P.J. Plauger: The Elements of Programming Style](https://en.wikipedia.org/wiki/The_Elements_of_Programming_Style)
